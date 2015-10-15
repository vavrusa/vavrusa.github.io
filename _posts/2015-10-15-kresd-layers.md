---
published: true
---


# Scripting in Knot DNS Recursive

This week I was approached by a man dressed in a duck costume, he asked me: *"These layers and modules you talk about, they're cool. But can it be even better?"*. After the initial costume distraction wore off, I pondered a bit and said: *"Sure, let me just grab a cup of coffee"*. The real story is that layers are now much more interactive, and the documentation is improved.

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
$ kresd -v
> modules = { 'example' }
> resolve('abcd.kiwi', kres.type.AAAA)
...
> example.count
1
```

This is only *passive* observation of the resolver I/O. You can also introspect status of the queries. For example - *"which are answered from cache?"* We can write something along the lines of:

```lua
req = kres.request_t(req)
local query = req:current()
if query.flags and kres.query.CACHED then
    mod.count = mod.count + 1
end
```

At this point, you should skim through the [Lua API reference](http://knot-resolver.readthedocs.org/en/latest/lib.html#apis-in-lua). The examples often reference to constants like `kres.type.AAAA` or accessors `req:current()`, the reference explains where to find what.

## Chaining queries

The previous example has shown how to observe the resolution chain. Let's see how we can change it. For example - for every NS record, I want its SOA as well. One way how to do that is to simply chain another query. Here's a [documentation reference](http://knot-resolver.readthedocs.org/en/latest/lib.html#for-developers).

Short version is that there is a *driver* behind resolution which does I/O, figures out when to ask what, sets correct flags and so on. The plus is that layers don't have to know about DNSSEC, caching layers, correct ordering etc. 

```lua
if pkt:qtype() == kres.type.SOA then
    local next = req:push(pkt:qname(), kres.type.NS, kres.class.IN)
    next.flags = kres.query.AWAIT_CUT + kres.query.DNSSEC_WANT
    return kres.DONE
else
	return state
end
```

This piece of code fetches `NS` for each `SOA` query, after the query completes. You can check that the record was fetched in the `finish` layer afterwards. This is convenient if you want to simply chain queries.

## Reordering queries

States signalize the outcome of layer pass-through, there are 3 interesting ones: `DONE`, `FAIL` and `YIELD`. A common idiom is to copy input state if you're not interested in this packet.

```lua
consume = function (state, req, pkt)
    if state == kres.FAIL then
        return state
    end
    return kres.DONE
end
```

The `YIELD` pauses current layer, starts solving whatever queries you pushed, and then resumes the paused layer. One example is DNSSEC, where you need a DNSKEY to verify signatures. Or use only `NS` records that have a `SOA`.

```lua
if state == kres.YIELD then
	local last = req:resolved()
	if bit.band(last.flags, kres.query.RESOLVED) then
		return kres.DONE -- Fetched NS
    else
    	return kres.FAIL -- Failed to fetch NS
    end
else
    if pkt:qtype() == kres.type.SOA then
        local next = req:push(pkt:qname(), kres.type.NS, kres.class.IN)
        next.flags = kres.query.AWAIT_CUT + kres.query.DNSSEC_WANT
        next.parent = qry
        return kres.YIELD -- Resolve NS and resume
    end
    return state
end
```

The first branch is executed when the paused layer resumes, there we can check whether the `NS` query finished successfuly. The second branch pushes the `NS` query, and then pauses.

## Rewriting queries

I'm a bit reluctant to write this, as there is not any *good* use case that comes to my mind. But let's say you want to sinkhole several queries, e.g. if the `QNAME` matches a pattern, we rewrite it to something else. This can be done before *driver* starts issuing queries, in the *begin* layer.

```lua
begin = function(state, req)
    req = kres.request_t(req)
    local query = req:current()
    if query:name():find('\4kiwi') then
    	req:pop(query) -- Pop old query, and replace with new one
        req:push('\7blocked', query.type, query.class, query.flags, 0)
    end
    return state
end
```

## Wrap up

That's it. Here we are with the examples for three different layer usage patterns in a few lines of Lua code. The advantage here is that you can drop custom modules in the configuration directory, no need to patch or recompile the software. If you don't mess up the Lua code, it's going to run close to C code speed. When in trouble achieving that, check the [Embedding LuaJIT in 30 minutes](http://en.blog.nic.cz/2015/08/12/embedding-luajit-in-30-minutes-or-so/) I wrote before.
