---
published: false
---


## Embedding LuaJIT 101 (and how to make most of it)

Since you're reading this, you probably know [Lua][lua-infuriating], probably the world's most infuriating language. If not, hop on to [Lua in 15 minutes][lua-15min] to get the basics right. Now there are two types of use cases where Lua shines - as a tiny script engine or configuration language, and as a high-performance filtering language (with JIT), I went through both of them with [kresd][kresd], so here are my notes.

### So what's it all about?

There is a well written book [Programming in Lua][pil-config] that covers everything from the syntax to C/Lua interfacing. If you're starting with Lua, chances are you're going to often use it as a reference. I'm just going to sum up the basics real quick.

Any Lua flavour will do for now, you can pick from [eLua][elua] for embedded, vanilla interpreter [Lua][lua] or [LuaJIT][luajit]. All of them compile quickly to a small (static or shared) library. Vanilla is about 200K, LuaJIT takes about twice as much. I'd recommend going with LuaJIT right off the bat, especially if you have plans for accessing C world from Lua. [Installation][luajit-install] is a cinch:

```bash
$ make && sudo make install
```
Let's make a mockup C application to fire up the VM and read out some values.

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

And compile it (add `-pagezero_size 10000 -image_base 100000000` to `cc` for [LuaJIT on OS X][luajit-install], section "Embedding LuaJIT").

```bash
$ cc $(pkg-config --cflags --libs luajit) luatest.c -o luatest
$ cat test.lua 
address = '255.255.255.255'
port = 67
$ ./luatest test.lua
address: 255.255.255.255, port: 67
```

Awesome! It reads our configuration with just few lines of code. There are 4 bread & butter interactions
with the Lua world that you can see here.

* (1) is calling Lua to execute a chunk of code (a *protected* call)
* (2) getting Lua values on Lua stack
* (3) reading out Lua values from Lua stack
* (4) stack manipulation

The [stack][lua-stack] is a bit difficult to wrap your head around first, but it boils down to this: stack top, the last pushed value, is always -1. If we push something, it becomes the new stack top. If we call a function, it unwinds the stack and pushed elements become arguments (or upvalues). Once again - pushed values are negative, function arguments positive.

If you're still not sure how this works, have a look at another ["embedding example"][embed-example] or play around a little bit with it. Links that you're going to find useful:

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

Let's try it out then with a simple function that returns the lenght of the argument.

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
and `YIADDR` the address that we're going to lease. You can find the whole code in this gist.

<script src="https://gist.github.com/vavrusa/7c4b02ac2e4714358273.js"></script>

Let's write some server logic in the Lua world now. Let's say we want to remember the leases of various clients,
and create new leases for unknown clients. We also want to do some ACLs on hardware address, because Lua is fun.

```lua
address = '127.0.0.1'
port = 53
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

Napkin calculations tell me this is fairly quick, but nowhere near the C speed. Let's have a look at the debugging and profiling facilities of LuaJIT.

### It's feels slow, what should I do?

The bad news is that idiomatic code may or may not be predictably fast like in compiled languages. The good news is that LuaJIT has excellent facilities to find out the cause.

#### Turning off JIT

There might be a problem with JITing itself, first thing you can do is to disable it and observe.

```bash
$ echo "jit.off()" >> test.lua
$ time ./luatest test.lua
real	0m0.603s
```

Wow, that's roughly half the time it took with JIT enabled. There are two diagnostic tools available - `jit.v` and `jit.dump`.

#### Running JIT in verbose mode

Module `jit.v` is able to print events happening in the JIT to either a file or stdout. It report trace aborts, transitions or any other obstructions to that toasty traces. You can enable it with a simple require, first re-enable the JIT that we turned off and run verbose.

```bash
$ echo "require('jit.v').start()" >> test.lua
$ time ./luatest test.lua
[TRACE flush]
[TRACE flush]
[TRACE flush]
[TRACE flush]
[TRACE flush]
...
```

That's a lot of trace flushes and the reason why it's so slow. It also happens somewhere out of Lua scope, so it must be in our C code. The problem here is that we push metatables to userdata objects, forcing the JIT to flush the code cache. Another common problem is doing [NYI operations][luajit-nyi] which cause trace aborts. This is especially costly, as the 

[lua-infuriating]: http://www.slideshare.net/jgrahamc/lua-the-worlds-most-infuriating-language
[lua-15min]: http://tylerneylon.com/a/learn-lua/
[luajit]: http://luajit.org/download.html
[lua]: http://www.lua.org/download.html
[elua]: http://www.eluaproject.net/get-started/downloads
[luajit-install]: http://luajit.org/install.html
[lua-stack]: http://www.lua.org/pil/24.2.html
[lua-cwrap]: https://john.nachtimwald.com/2014/07/12/wrapping-a-c-library-in-lua/
[embed-example]: http://lua-users.org/wiki/SimpleLuaApiExample
[lua-users]: http://lua-users.org/wiki/TutorialDirectory
[lua-yoyo]: http://pgl.yoyo.org/luai/i/lua_pcall
[pil-registry]: http://www.lua.org/pil/27.3.1.html
[pil-metatables]: http://www.lua.org/pil/13.html
[dhcp]: https://tools.ietf.org/html/rfc2131
[dhcp-msg]: https://tools.ietf.org/html/draft-ietf-dhc-implementation-02#page-25
[dhcp-fields]: https://tools.ietf.org/html/draft-ietf-dhc-implementation-02#page-29
[luajit-nyi]: http://wiki.luajit.org/NYI
[luajit-pcall-overhead]: http://lua-users.org/lists/lua-l/2006-10/msg00369.html
[luajit-shortcall]: http://lua-users.org/lists/lua-l/2011-02/msg00421.html