---
layout: post
title:  "Metadata Cache Consistency and Performance"
date:   2022-02-25 16:00:00
published: false
---
### Overview
Problem: 
1. API service needs huge local metadata cache (for multi-tenancy, 1000 copies of metadata, very few static part to share)
2. the metadata could change
3. when changed, all services should refresh timely but without rush into the database to load metadata.



