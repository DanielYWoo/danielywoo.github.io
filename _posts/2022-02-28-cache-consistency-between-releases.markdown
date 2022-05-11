---
layout: post
title:  "Cache Consistency Between Releases"
date:   2022-02-28 18:00:00
published: false
---



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
solution: 

phase 1, deploy new code and decomission old nodes gradually, allow new node write to cache with the new field, but disable reading of the new field. wait for cache to expire. then all  so far, if the new field is non-null, all cached data should has the new value in the new data schema. If the new field is nullable, the new code knows it must be null, instead of an uncached field.
phase 2, enable the null check. use the feature toggle to enable UI display and Open API.

 the new field in UI, and the new code keeps the old code, run new code if not null, run old code if null. Only after all service instances are deployed, wait for the od cache expires, verify all cached data has the new fields and works, use the feature toggle to enable UI display and the new field. Deploy it again, if the new field is not-null.

new fields must be nullable. if the new value is to be non-null, 

solution: version in key, only update the version for changed entities. But this introduces two versions of cached data, inconsistency in UI, client sees different values when requests is routed in round-robin manner or randomly. e.g., 

v1 writes Person(id=1,name=jack,salary=100)
v1 writes Person(id=1,name=jack,salary=200) (will not update the cached value of v1)

client reads Persion(id=1) twice, the first request goes to v1, the second goes to v2 and the third goes to v1 again.
solution: very difficult in canary, even sticky user won't work. Remove old nodes causes user migration to new nodes and see inconssistency. When a node is removed, clear both old and new cached data can help, but this is very complicated to implement.


## solutions



