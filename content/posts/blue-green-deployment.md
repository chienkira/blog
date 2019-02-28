---
title: "Blue Green Deployment"
date: 2019-02-25T15:07:37+09:00
draft: true
tags: []
language: vietnamese
---

```bash
                            S3 <--> API G <----> 
                            ↑                   ↓
Client <--> Cloudfront <----                   DB 
            ↓  ↑            ↓                   ↑
            ↓  ↑            S3 <--> API G <-->
         Lambda@edge^

```