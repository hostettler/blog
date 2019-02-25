---
layout: post
title: "Cleaning Intermediate Docker Images"
date: 2015-12-02 01:56:34 +0100
comments: true
categories: Docker
---
I recently started to use Docker. It is a great tool that significantly increases developer's productivity. However, I regularly encounter disk space problems when developing new images. Indeed, I sometimes end up with dangling images and containers. Hereafter, a simple script that cleans up most of them.

```bash
#!/bin/bash
docker rmi -f `docker images | grep "^<none>" | cut -c41-52`
docker rmi $(docker images -q -f dangling=true)
docker rm -v $(echo $(docker ps -q --no-trunc) $(docker ps -a -q --no-trunc) | sed 's|\s|\n|g'  | sort | uniq -u)
```
