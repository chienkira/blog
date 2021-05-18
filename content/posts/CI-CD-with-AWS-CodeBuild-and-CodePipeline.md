---
title: "CI/CD With AWS CodeBuild and CodePipeline - Part 1"
date: 2021-05-17T22:47:21+09:00
draft: true
tags: [aws, devops]
language: vietnamese
toc: true
authors: [chienkira]
cover: /blog/images/xxx.png
---

**Bài viết dự tính gồm có 2 phần (đây là part 1) sẽ chia sẻ đến mọi người cách cài đặt CI/CD trong thực tế sử dụng dịch vụ AWS CodeBuild và CodePipeline. Hi vọng trở thành thông tin tham khảo cho những bạn đang xem xét sử dụng dịch vụ CI/CD của AWS hay đang vướng mắc trong quá trình thiết kế nhé.**

## Giải thích sơ sơ về CodeBuild và CodePipeLine

Nằm trong bộ các developer tools, cùng với các dịch vụ khác như CodeCommit vân vân có thể thấy tham vọng lớn lao của AWS - tạo ra sân chơi trang bị tận răng cho các developers. Sẽ không cần thiết phải sử dụng đến bất kì dịch vụ nào khác cũng có thể host source repo, cài đặt tự động tích hợp (CI) tự động deploy (CD) vân vân mây mây. Nghe rất ấn tượng nhưng cũng phải review thật tâm rằng là, lần đầu sử dụng mình thấy nó chưa hoàn thiện nếu so sánh với các dịch vụ khác mình đã từng trải nghiệm ví dụ như CircleCI. Nếu bạn là người có kinh nghiệm Devops, kinh qua Jenkins, CircleCI rồi thì chắc sẽ thấy hơi khó sử dụng đấy, nhưng mà lợi ích từ việc dễ dàng liên kết với các dịch vụ khác trong hệ sinh thái của AWS thì cũng khá là lớn!

1. dịch vụ CodeBuild

2. dịch vụ CodePipeline

3. quan hệ giữa 2 dịch vụ CodeBuild và CodePipeline

## Tổng thể kiến trúc CI/CD


### Cài đặt CI với CodeBuild

Phần 1 này tập trung vào phần CI, sẽ hướng dẫn bạn cách tạo build project trong CodeBuild để tự động hoá việc đóng gói/tích hợp.
