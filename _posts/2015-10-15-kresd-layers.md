---
published: false
---

# Scripting in Knot DNS Recursive

This week I was approached by a man dressed in a duck costume, he asked me: *"These layers and modules you talk about, they're okay. But can you make it even better?"*. After the initial costume distraction wore off, I pondered a bit and said: *"Sure, let me just grab a cup of coffee"*.

## What are layers

Callbacks you can execute on each new/completed query, and resolver sub-requests. For example - log all `AAAA` answers from `*.kiwi`, or refuse all queries matching a blocklist. Here's an example similar to [the documentation](http://knot-resolver.readthedocs.org/en/latest/modules_api.html#writing-a-module-in-lua).

```lua
local mod = {}
mod.count = 0
mod.layer = {
	consume = function (state, req, pkt)
	  pkt = kres.pkt_t(pkt)
		if pkt:qtype() == kres.type.AAAA and pkt:qname():find('\4kiwi') then
		  mod.count = mod.count + 1
		end
		return state
	end
}
return mod
```

Save it as `example.lua` and start `kresd`.

```bash
$ kresd
> modules = { 'example' }
$ dig @localhost AAAA whois.kiwi
```

Now you can check the counter in CLI.

```bash
> example.count
1
```

This is only *passive* observation of the resolver I/O. You can also introspect status of the queries. For example - *"which are answered from cache?"* There is now a [Lua API reference](http://knot-resolver.readthedocs.org/en/latest/lib.html#apis-in-lua) that you can use. So we can write something along the lines of:

```
	req = kres.request_t(req)
    local query = req.current()
    if query.flags and kres.query.CACHED then
    	mod.count = mod.count + 1
    end
```

## What are layers






