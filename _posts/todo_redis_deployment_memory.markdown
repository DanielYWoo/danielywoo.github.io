---
layout: post
title:  "Redis Memory Usage Optimization"
date:   2015-10-28 13:11:45
published: false
---


jemalloc
RSS > commit, pool
overcommit
pointer 32

# Compatibility
It's important to avoid rebuilding cache, espcially for non-stop applications with canary deployment model.

## protocol compatiblity:
Binary protocols like kryo:
- kryo major version upgrade, no way
- add fields, see how protocol buffers support it

## data compatiblity:
v1 code write Person(id=1,name=jack)
v2 code write Person(id=1,name=jack,salary=100)

v1 writes first then v2 read, - (how can v2 knows salary is not cached instead of null or zero)
even with entity version, it's inefficient. e.g.,
v2 writes first then v1 updates, v2 reads and detects lower version, rebuild the cache is too expensive.
solution: new fields must be nullable, use a feature toggle to disable displaying the new field in UI, and skip new business logic on the new field if null. Only after all service instances are deployed, wait for the od cache expires, verify all cached data has the new fields and works, use the feature toggle to enable UI display and the new field. Deploy it again, if the new field is not-null.


solution: version in key, only update the version for changed entities. But this introduces two versions of cached data, inconsistency in UI, client sees different values when requests is routed in round-robin manner or randomly. e.g., 

v1 writes Person(id=1,name=jack,salary=100)
v1 writes Person(id=1,name=jack,salary=200) (will not update the cached value of v1)

client reads Persion(id=1) twice, the first request goes to v1, the second goes to v2 and the third goes to v1 again.
solution: very difficult in canary, even sticky user won't work. Remove old nodes causes user migration to new nodes and see inconssistency. When a node is removed, clear both old and new cached data can help, but this is very complicated to implement.


## solutions



