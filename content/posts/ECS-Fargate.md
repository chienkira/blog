---
title: "Memo about AWS ECS Fargate"
date: 2020-01-09T17:50:27+09:00
draft: no
tags: [aws, infra, docker, ECS]
language: vi
toc: false
authors: [chienkira]
---

**Mới đầu năm 2020 nhưng lại có cơ hội được tham khảo một hệ thống xài docker container vào môi trường prod. Ở đây họ dùng ECS và Fargate của AWS. Mình vẽ lại sơ đồ cấu thành của nó lưu lại làm tham khảo và ôn tập lại ECS luôn.**

## Cấu thành của 1 hệ thống API dùng ECS Fargate

↓ Ảnh svg, có thể tải về và phóng to ra.

![ECS Fargate](/blog/images/ECS_Fargate.svg)