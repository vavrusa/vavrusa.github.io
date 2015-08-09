---
published: false
---


## Embedding LuaJIT in 30 minutes (or so)

Since you're reading this, you probably know [Lua][lua-infuriating], probably the world's most infuriating language. If not, hop on to [Lua in 15 minutes][lua-15min] to get the basics right. Now there are two types of use cases where Lua shines - as a tiny script engine or configuration language, and as a high-performance filtering language (with JIT), I went through both of them with [kresd][kresd], so here are my notes.

### So what's it all about?

There is a well written book [Programming in Lua][pil-config] that covers everything from the syntax to C/Lua interfacing. If you're starting with Lua, chances are you're going to often use it as a reference. I'm just going to sum up the basics real quick.

Any Lua flavour will do for now, you can pick from [eLua][elua] for embedded, vanilla interpreter [Lua][lua] or [LuaJIT][luajit]. All of them compile quickly to a small (static or shared) library. Vanilla is about 200K, LuaJIT takes about twice as much. I'd recommend going with LuaJIT right off the bat, especially if you have plans for accessing C world from Lua. [Installation][luajit-install] is a cinch:

```bash
$ make && sudo make install
```
Let's make a mock-up C application to fire up the VM and read out some values.

```c
#include <stdio.h>
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>

/* Convenience stuff */
static void close_state(lua_State **L) { lua_close(*L); }
#define cleanup(x) __attribute__((cleanup(x)))
#define auto_lclose cleanup(close_state) 

int main(int argc, char *argv[])
{
	/* Create VM state */
	auto_lclose lua_State *L = luaL_newstate();
	if (!L)
		return 1;
	/* Load config file */
	if (argc > 1) {
		luaL_loadfile(L, argv[1]); /* (1) */
		int ret = lua_pcall(L, 0, 0, 0);
		if (ret != 0) {
			fprintf(stderr, "%s\n", lua_tostring(L, -1));
			return 1;
		}
	}
	/* Read out config */
	lua_getglobal(L, "address"); /* (2) */
	lua_getglobal(L, "port");
	printf("address: %s, port: %ld\n", /* (3) */
		lua_tostring(L, -2), lua_tointeger(L, -1));
	lua_settop(L, 0); /* (4) */
	return 0;
}
```

There are four bread & butter interactions with the Lua world that you can see here:
*(1)* is calling Lua to execute a chunk of code
*(2)* getting Lua values on Lua stack
*(3)* reading out Lua values from Lua stack
*(4)* stack manipulation

Let's go straight for the test phase. Don't forget to add `-pagezero_size 10000 -image_base 100000000` for [LuaJIT on OS X][luajit-install], see section *"Embedding LuaJIT"*).

```bash
$ cc ldhcp-ex1.c $(pkg-config --cflags --libs luajit) -lm -ldl -o luatest
$ cat test.lua 
address = '255.255.255.255'
port = 67
$ ./luatest test.lua
address: 255.255.255.255, port: 67
```

*Mission All Clear*. It reads our configuration with just a few lines of code. 

Back to the example, the [stack][lua-stack] is the difficult bit to wrap your head around first, but it boils down to this: stack top, the last pushed value, is always -1. If we push something, it becomes the new stack top. If we call a function, it unwinds the stack and pushed elements become arguments (or [upvalues][lua-upval]). Once again - pushed values are negative, function arguments positive.

If you're still not sure how this works, have a look at another ["embedding example"][embed-example] guide, or play around a little bit with it. Here are links that you might find useful:

* [Lua users tutorials][lua-users] - various examples and code snippets, three stars out of five.
* [Lua reference][lua-yoyo] - well put together API reference for both Lua and Lua/C APIs, four and a half stars.

### Extending C application with Lua

You can call functions the same way as you interact with variables. Now that you've mastered the stack, your first idea would be to do something like this:

```c
static int on_recv(lua_State *L, char *buf, size_t len)
{
	lua_getglobal(L, "callback");
	lua_pushlstring(L, buf, len); /* Binary strings are okay */
	int ret = lua_pcall(L, 1, 1, 0); /* 1 argument, 1 result */
	printf("ret: %d, buflen: %ld\n", ret, lua_tointeger(L, -1));
	lua_pop(L, 1);
	return ret;
}
...
	/* Call a callback for mock datagram. */
	char buf[512] = { 0x05, 'h', 'e', 'l', 'l', 'o' };
	on_recv(L, buf, sizeof(buf));
```

Let's try it out then with a simple function that returns the length of the argument.

```bash
$ cat test.lua
function callback(buf)
	return #buf
end
$ ./luatest test.lua
address: 255.255.255.255, port: 67
ret: 0, buflen: -> 512 <-
```

At this point, you should start to be worried about the tiny inefficiencies. Looking up globals is relatively expensive,
we can look later how much, but fortunately Lua has a [fast registry][pil-registry] for storing various values.

```c
static int on_recv(lua_State *L, int cb_ref, char *buf, size_t len)
{
	lua_rawgeti(L, LUA_REGISTRYINDEX, cb_ref);
	lua_pushlstring(L, buf, len);
	int ret = lua_pcall(L, 1, 1, 0);
	printf("ret: %d, buflen: %ld\n", ret, lua_tointeger(L, -1));
	lua_pop(L, 1);
	return ret;
}
...
	lua_getglobal(L, "callback");
	int cb_ref = luaL_ref(L, LUA_REGISTRYINDEX);
	on_recv(L, cb_ref, buf, sizeof(buf));
```

This is especially helpful if you call the function repeatedly.

### Extending Lua code with C

Our previous example has one problem, it has no purpose. So we'll give it one, we're going to write a tiny [DHCP][dhcp] server. DHCP is one of the Internet's most boring and yet widely used protocols whose purpose is to assign IP addresses to clients.

The messages in DHCP [are binary][dhcp-msg]. Strings in Lua are immutable, so we can't just fiddle with random bytes in the packet, but we're going to call the C functions to do it for us. There are two ways to call into C - Lua/C interface and LuaJIT FFI. The former is a-okay, the latter rocks.

The Lua/C interface can provide enough sugar coating, so that C userdata can look like a regular Lua object.

```c
static int msg_op(lua_State *L)
{ /* Get/set opcode */
	char *msg = lua_touserdata(L, 1);
	if (lua_isnumber(L, 2)) {
		msg[0] = lua_tointeger(L, 2) & 0xFF;
	}
	lua_pushnumber(L, msg[0]);
	return 1;
}
/* Register 'msg' meta table */
static int msg_register(lua_State *L)
{
	static const luaL_Reg wrap[] = {
		{ "op",     &msg_op },
		{ NULL, NULL }
	};
	luaL_newmetatable(L, "msg");
	lua_pushvalue(L, -1);
	lua_setfield(L, -2, "__index");
	luaL_openlib(L, NULL, wrap, 0);
	lua_pop(L, 1);
	return 0;
}
```

You can then apply [metatables][pil-metatables] to provided C userdata argument.

```c
static int on_recv(lua_State *L, int cb_ref, char *buf, size_t len)
{
	lua_rawgeti(L, LUA_REGISTRYINDEX, cb_ref);
	lua_pushlightuserdata(L, buf);
	luaL_getmetatable(L, "msg");
	lua_setmetatable(L, -2);
	return lua_pcall(L, 1, 1, 0);
}
```

From now, you can call the methods on the `buf` object from the Lua world, as if it was native.

```lua
function callback(buf)
	print(buf:op())
	return 0
end
```

I have extended this example with additional methods for access to [YIADDR and CHADDR][dhcp-fields]
fields in the DHCP message, so we have something to work with. `CHADDR` represents client's hardware address,
and `YIADDR` the address that we're going to lease. You can find the whole code [in this gist](https://gist.github.com/vavrusa/7c4b02ac2e4714358273).

With this we can write some server logic in the Lua world now. Let's say we want to remember the leases of various clients,
and create new leases for unknown clients. We also want to do some ACLs on hardware address, because Lua is fun.

```lua
address = '255.255.255.0'
port = 67
-- Only allow hwaddr with these prefixes
local allowed_prefix = { '\0\0', '\5\3' }
local function acl(hwaddr)
    for i,v in ipairs(allowed_prefix) do
        if hwaddr:sub(0, #v) == v then
            return true
        end
    end
    return false
end
-- Flip some bits in the buffer and return length
local lease = {}
function callback(buf, len)
    if buf:op() == 0x01 then -- DHCPDISCOVER
        buf:op(2) -- DHCPOFFER
        local hwaddr = buf:chaddr()
        if not acl(hwaddr) then return 0 end
        local client_ip = lease[hwaddr]
        if client_ip then -- keep leased addr
            buf:yiaddr(client_ip)
        else -- stub address range (:
            client_ip = '192.168.1.1'
            buf:yiaddr(client_ip)
            lease[hwaddr] = client_ip
        end
    end
    return len
end
```

That's a pretty straightforward piece of code - few branches, and a short loop.
Lua likes `local` variables where possible, one transgression is the overuse of metatables to keep the code short.
Let's put it under a very crude benchmark with a loop of 1000000 packets to see how it goes.

```bash
$ time ./luatest test.lua
real	0m1.274s
```

Napkin calculations tell me this is "okay quick", but nowhere near the C speed.

### It's feels slow, what should I do?

The bad news is that idiomatic code may or may not be predictably fast like in the compiled languages, which makes Lua sometimes so infuriating. Alas *"engineering"* in the software engineering is about understanding what's going on, and the good news is that LuaJIT has excellent facilities to assist you with finding the root cause. Usually, there is something preventing trace compilation or excessive garbage collection.

First thing you can do is to disable JIT and observe.

```bash
$ echo "jit.off()" >> test.lua
$ time ./luatest test.lua
real	0m0.603s
```

Wow, that's roughly half the time it took with JIT enabled. There are two diagnostic tools available - `jit.v` and `jit.dump`.
I'm going to start with the first one, as the latter one falls into *"profiling"* in my mind. Notice that despite the absence of
trace compiler, it's already churning close to 1.5M calls per second.

### Tracing the tracer

Module `jit.v` is able to print events happening in the JIT to either a file or stdout. It reports trace aborts, transitions or any other obstructions to that toasty compiled traces. You can enable it with a simple `require('jit.v').start()`.
First re-enable the JIT that we turned off, and restart in verbose mode.

```bash
$ echo "require('jit.v').start()" >> test.lua
$ time ./luatest test.lua
[TRACE flush]
[TRACE flush]
[TRACE flush]
[TRACE flush]
[TRACE flush]
... bazillion times ...
```

That's a lot of trace flushes and hence reason why it's so slow. It also happens somewhere out of Lua scope, so it must be in our C code. The problem here is that we push metatables to userdata objects, forcing the JIT to flush the code cache. Another common problem is doing [NYI operations][luajit-nyi] which cause trace aborts. This is especially costly, as the tracer does a lot of work in background which never comes to fruition.

Now we're left with two choices, rewrite our C API without metatables - e.g. `buf:op()` would become `dhcp.op(buf)`, or rewrite it using LuaJIT FFI. First choice is portable between different Lua flavours, but Lua/C calls would still abort traces (as [they're NYI][luajit-nyi]). The second choice locks us to LuaJIT unfortunately. Since this is a tutorial, I'm going to show you how to rewrite the current bindings using FFI.

A short note on portability - Lua usually isn't backwards-compatible between minor versions. If you need to target multiple versions, you usually end up with an `#ifdef` boilerplate. LuaJIT targets 5.1 ABI with [several extensions][luajit-extensions].
For example, 5.2 introduced proper scoping, that changes a way of [doing sandboxing][knot-sandbox1].

### FFI bindings to your C code

The idea behind FFI is to get rid of the Lua/C boilerplate code and [describe your datastructures][luajit-ffi] and function signatures in a mix of C definitions and Lua. The C part could look like this.

```lua
local ffi = require('ffi')
ffi.cdef[[
/* DHCP header format */
struct __attribute__((packed)) dhcp_msg {
	uint8_t op;     /* Header */
	uint8_t htype;
	uint8_t hlen;
	uint8_t hops;
	uint32_t xid;
	uint16_t secs;
	uint16_t flags; /* Address fields */
	char _ciaddr[4], _yiaddr[4], _siaddr[4], _giaddr[4];
	char _chaddr[16];
};
/* Constants */
struct dhcp_op {
	static const int DISCOVER = 0x01;
	static const int OFFER    = 0x02;
};
/* C functions */
void yiaddr_set(char *msg, const char *yiaddr);
]]
```

This describes the structure of the packet, constants and a C function signature. Notice how
FFI describes [constants/enums using a structure][luajit-const] with static const integers, compared to idiomatic C `enum { ... }`.
Now we can cast the C string buffer (in form of Lua light userdata) to the structure type.

```lua
local msg = ffi.cast('struct dhcp_msg *', buf)
print(msg.htype)
```

LuaJIT can also assign metatables to C types, we can use this to give the message nicely wrapped operations.
They're also compiled, so it's very very cheap.

```lua
local dhcp_msg_t = ffi.typeof('struct dhcp_msg')
ffi.metatype( dhcp_msg_t, {
	__index = {
		yiaddr = function(msg, addr)
			ffi.C.yiaddr_set(msg, addr)
		end,
	},
})
local msg = ffi.cast('struct dhcp_msg *', buf)
print(msg:chaddr('192.168.1.1'))
```

The `ffi.C` points to global C namespace, kind of like `dlsym(RTLD_DEFAULT)`. You can however call to other libraries
using `ffi.open()`, see the [FFI Tutorial][luajit-ffitut], section "Accessing the zlib Compression Library".

We have created [three files][ldhcp-ex2], one is [our C application][ldhcp-ex2-c], the second is a [Lua module with FFI bindings][ldhcp-ex2-ffi], and finally [our userscript][ldhcp-ex2-lua]. I have created gists for all three, so you can have a look at them and get it ready for the starting blocks.

### Inspecting traces for profit

```bash
$ time ./luatest test.lua
real	0m1.064s
```

That looks faster than the previous run with the JIT on, but it's still slow. Let's start with verbose mode to see what's wrong.

```bash
[TRACE --- test.lua:8 -- leaving loop in root trace at test.lua:13]
    for i,v in ipairs(allowed_prefix) do
```

Oh. It looks like that in Lua 2.1 `ipairs()` is [already compiled][luajit-nyi], but the short loop is causing aborts.
To investigate closer, we can require the `jit.dump` module to show us traces, IR and compiled code events. [Here's][luajit-dump] a comment
on all available options. Since the log can be rather large, we can choose to print it to a separate file instead of `stdout`.

```lua
$ echo "require('jit.dump').start('bsx', 'jit.log')" >> test.lua
$ time ./luatest test.lua
real	0m1.064s
$ less jit.log
...
---- TRACE 4 start test.lua:7
0001  GGET     1   0      ; "ipairs"
0002  UGET     2   0      ; allowed_prefix
0003  CALL     1   4   2
0000  . FUNCC               ; ipairs
0004  JMP      4 => 0014
0014  ITERC    4   3   3
0000  . FUNCC               ; ipairs_aux
0015  JITERL   4   1
---- TRACE 4 abort test.lua:9 -- inner loop in root trace
```

I'm not entirely sure why this aborts here. I suspect it's because of the `ipairs()` iteration with [`ITERC`][luajit-iterc]
over a very short loop (2 items). Let's see if we can help it by rewriting it to for loop *(1)*.
Another thing we can do is to get rid of the `return` in the loop body using `break` *(2)*, so that the trace never ends
inside the loop.

```lua
local function acl(hwaddr)
    local ret = false
    for i = 1, #allowed_prefix do -- (1)
        local v = allowed_prefix[i]
        if hwaddr:sub(0, #v) == v then
            ret = true
            break -- (2)
        end
    end
    return ret
end
```

And try it again.

```bash
$ time ./luatest test.lua
real	0m0.135s
```

Whoa, this is an order of magnitude faster and `jit.dump` shows that it succeeded to find a trace for the whole callback.
Usually there are a few aborted traces before it finds it, it's nothing to worry about.

*Note* - Branches are not allowed in traces, so the compiler deals with it by spawning side traces. The [LuaJIT performance guide][luajit-perfguide] states that highly biased are *okay*, as the most likely branch is going to get compiled in the root trace.
However here we can't presume anything about the branching. Have a look at the ["Tracing JITs and modern CPUs part 3"][luajit-badcase] for more information.

Since we're close to the **8M** callbacks/s figure, we can have a look at the profile to fine tune the remaining slowdowns.

### Finding hotspots with LuaJIT profiler

Mike Pall [introduced][luajit-p] the sampling profiler for the 2.1, which I highly recommend for the performance improvements only.
The sampling rate is configurable and it's really quick. Let's have a look at it first.

```bash
$ echo "require('jit.p').start('vl')" >> test.lua
$ ./luatest test.lua
80%  Compiled
  -- 50%  dhcp.lua:35
  -- 38%  test.lua:9
  -- 13%  test.lua:21
20%  C code
  -- 100%  test.lua:21
```

The [option `v`][luajit-popts] stands for *"show VM states"*, in this case *"Compiled"* or *"C code"*, and `l` for per-line dumps.
Now we know that approximately 80% of the time is spent in the toasty compiled traces and 20% in C callbacks.
Since it's sampling our already short callbacks, it may not be accurate. We can improve the resolution of the sampling with `i` option.

```bash
$ echo "require('jit.p').start('vli1')" >> test.lua
$ ./luatest test.lua
76%  Compiled
  -- 42%  test.lua:9
  -- 31%  dhcp.lua:35
  -- 15%  test.lua:21
  --  8%  test.lua:22
  --  4%  dhcp.lua:53
18%  C code
  -- 100%  test.lua:21
 6%  Interpreted
  -- 100%  test.lua:21
```

It looks that the message type conversion (`test.lua:21`) took some time to compile, the other is looping.
Nothing that stands out. We could probably unroll the loop since it's short or rewrite the `yiaddr_set` in Lua, so it
could be inlined. But that wouldn't help it dramatically.
We're almost at the limit of [`pcall() / sec`][luajit-pcall-overhead].

### Breaking the ceiling

The `pcall() / sec` overhead is going to be the limit if you use C to Lua callbacks. On my computer it's about 50ns,
that's a budget for 20 million calls per second. To give it some sense of scale, framing rate of 10GbE is 14.88M pps.
Multithreaded web servers clock up to several millions of requests per second with pipelining. Typical single-threaded UDP
daemons [do less][cf-1mpps] than that.

While that is not C call speed, it's competitive. After all, performance is about squeezing in a time budget required for the job.
That time budget is different for numerical simulations, where the cost of C/Lua call would dominate. More so, the JIT can't cross
a `pcall()` boundary so the trace can be only as long as one single callback.

The only way to break this limit is moving the event loop to Lua, and call back to helper functions in C. Sort of the other way round.
This is how the C event loop could be rewritten in Lua:

```lua
local len = 512
local buf = ffi.new('char[?]', len)
local dhcp_req = dhcp.msg_t(buf)
for i = 1, 1000000 do
    dhcp_req.op = dhcp.op.DISCOVER
    len = callback(buf, len)
end
```

Running it is indeed twice as fast, the JIT was able to compile the whole loop without any resumes, in or out.
The downside is that it's not always possible to rewrite the C event loop due to an already existing code base,
specific networking or custom memory allocators. Speaking of which.

#### On garbage collection

Lua is a garbage collected language. I realize that's a blow for a diehard C programmer whose manual memory management is infallible.
I'm not going to argue pros and cons here, but the truth is that I make mistakes. Moreover, a lot of people take `malloc / free` for an efficient memory management. In real world, the GC hasn't been an issue for me (performance-wise).
In addition, it offers [some tweaks][luatut-gc] for time-constrained applications. Up to a point though,
as it describes the constraints [very vaguely][lua-gc]:

> "The step multiplier controls the relative speed of the collector relative to memory allocation."

The only time I increased the step size was when the default GC speed couldn't keep up with the high packet rate,
but it can also be that the code generated too much [unnecessary garbage][lua-gctips].

### Further tips

This about covers the process of embedding LuaJIT in your C application. We went from zero to stub server doing 8 million callbacks per second, and peeked at the additional performance improvements. I sure hope it dispelled the worries about embedding a scripting language in your application. A general tip is to watch the [LuaJIT mailing list][luajit-mlist] for hidden gems and solutions. I tried to reference several of those in this article, but I'm sure something has eluded me.

I also hope that this might help to attract more people to LuaJIT. Mike Pall, the author, has announced plans to reduce his engagement in the project. While it is healthy and in [good hands][luajit-cf], it would be awesome if it kept the momentum. The Gotham still needs its true hero.

Further tips (subject to change):

* [Numerical Computing Performance Guide][luajit-perfguide]
* [Tracing JITs and modern CPUs part 3: A bad case][luajit-badcase]
* [high-performance packet filtering with pflua](https://wingolog.org/archives/2014/09/02/high-performance-packet-filtering-with-pflua)

[kresd]: https://github.com/CZ-NIC/knot-resolver
[lua-infuriating]: http://www.slideshare.net/jgrahamc/lua-the-worlds-most-infuriating-language
[lua-15min]: http://tylerneylon.com/a/learn-lua/
[luajit]: http://luajit.org/download.html
[lua]: http://www.lua.org/download.html
[elua]: http://www.eluaproject.net/get-started/downloads
[luajit-install]: http://luajit.org/install.html
[lua-upval]: http://www.lua.org/pil/27.3.3.html
[lua-stack]: http://www.lua.org/pil/24.2.html
[lua-cwrap]: https://john.nachtimwald.com/2014/07/12/wrapping-a-c-library-in-lua/
[embed-example]: http://lua-users.org/wiki/SimpleLuaApiExample
[lua-users]: http://lua-users.org/wiki/TutorialDirectory
[lua-yoyo]: http://pgl.yoyo.org/luai/i/lua_pcall
[pil-config]: http://www.lua.org/pil/25.html
[pil-registry]: http://www.lua.org/pil/27.3.1.html
[pil-metatables]: http://www.lua.org/pil/13.html
[dhcp]: https://tools.ietf.org/html/rfc2131
[dhcp-msg]: https://tools.ietf.org/html/draft-ietf-dhc-implementation-02#page-25
[dhcp-fields]: https://tools.ietf.org/html/draft-ietf-dhc-implementation-02#page-29
[luajit-nyi]: http://wiki.luajit.org/NYI
[luajit-pcall-overhead]: http://lua-users.org/lists/lua-l/2006-10/msg00369.html
[luajit-shortcall]: http://lua-users.org/lists/lua-l/2011-02/msg00421.html
[luajit-perfguide]: http://wiki.luajit.org/Numerical-Computing-Performance-Guide
[luajit-extensions]: http://luajit.org/extensions.html
[luajit-ffi]: http://luajit.org/ext_ffi_api.html
[luajit-ffitut]: http://luajit.org/ext_ffi_tutorial.html
[luajit-const]: http://wiki.luajit.org/ffi-knowledge#Constants
[knot-sandbox1]: https://github.com/CZ-NIC/knot-resolver/blob/0f7567d2dc1e94360d9473d03591858909a792c6/daemon/lua/sandbox.lua#L103
[ldhcp-ex2]: https://gist.github.com/vavrusa/926ced5fb2944da4fa66
[ldhcp-ex2-c]: https://gist.github.com/vavrusa/926ced5fb2944da4fa66#file-ldhcp-ex2-c
[ldhcp-ex2-ffi]: https://gist.github.com/vavrusa/926ced5fb2944da4fa66#file-dhcp-lua
[ldhcp-ex2-lua]: https://gist.github.com/vavrusa/926ced5fb2944da4fa66#file-test-lua
[luajit-dump]: https://github.com/luvit/luajit-2.0/blob/master/src/jit/dump.lua#L29
[luajit-abort]: https://www.freelists.org/post/luajit/simple-change-causes-trace-aborts,1
[luajit-iterc]: http://wiki.luajit.org/Bytecode-2.0#Loops-and-branches
[luajit-p]: http://permalink.gmane.org/gmane.comp.lang.lua.luajit/3413
[luajit-popts]: http://repo.or.cz/w/luajit-2.0.git/blob/8cc89332ffa3b65a43f6e730df18e282bb66ea41:/src/jit/p.lua#l22
[cf-1mpps]: https://blog.cloudflare.com/how-to-receive-a-million-packets/
[luatut-gc]: http://luatut.com/collectgarbage.html
[lua-gc]: http://www.lua.org/manual/5.1/manual.html#2.10
[lua-gctips]: http://lua-users.org/wiki/OptimisingGarbageCollection
[luajit-mlist]: http://luajit.org/list.html
[luajit-cf]: http://www.freelists.org/post/luajit/Looking-for-new-LuaJIT-maintainers
[luajit-badcase]: https://github.com/lukego/blog/issues/8
