---
published: false
---

## Embedding LuaJIT 101 (and how to make most of it)

Since you're reading this, you probably know [Lua][lua-infuriating], probably the world's most infuriating language. If not, hop on to [Lua in 15 minutes][lua-15min] to get the basics right. Now there are two types of use cases where Lua shines - as a tiny script engine / configuration language, and as a high-performance filtering language (with JIT), I went through both of them with [kresd][kresd], so here are my notes.

### Using it as a configuration language

Any Lua flavour will do for this, you can basically pick from [eLua][elua] for embedded, through vanilla interpreter [Lua][lua] to [LuaJIT][luajit]. All of them compile quickly to a small (static or shared) library. Vanilla is about 200K, LuaJIT takes about twice as much. If your plan is not to stay with a configuration, I'd recommend going with LuaJIT right off the bat. [Installation][luajit-install] is a cinch:

```bash
$ make && sudo make install
```


[lua-infuriating]: http://www.slideshare.net/jgrahamc/lua-the-worlds-most-infuriating-language
[lua-15min]: http://tylerneylon.com/a/learn-lua/
[luajit]: http://luajit.org/download.html
[lua]: http://www.lua.org/download.html
[elua]: http://www.eluaproject.net/get-started/downloads
[luajit-install]: http://luajit.org/install.html