---
title: "Am Hiểu Blue Green Deployment Trong 5 Phút"
date: 2019-02-25T15:07:37+09:00
draft: true
tags: [aws, sre, infra]
language: vi
toc: true
authors: [chienkira]
---

**Insert Lead paragraph here.**

## New Cool Posts

```bash
                            S3 <--> API G <----> 
                            ↑                   ↓
Client <--> Cloudfront <----                   DB 
            ↓  ↑            ↓                   ↑
            ↓  ↑            S3 <--> API G <-->
         Lambda@edge^

```