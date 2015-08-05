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
address = '127.0.0.1'
port = 53
$ ./luatest test.lua
address: 127.0.0.1, port: 53
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
lua_getglobal(L, "callback");
lua_pushlstring(L, buf, buflen);
int ret = lua_pcall(L, 1, 1, 0);
```

### Extending Lua code with C

The problem - you can now execute a callback in Lua, but 



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