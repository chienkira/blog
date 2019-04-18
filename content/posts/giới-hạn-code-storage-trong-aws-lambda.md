---
title: "Giới Hạn Code Storage Trong Aws Lambda"
date: 2019-04-15T17:37:58+09:00
draft: true
tags: []
language: vi
toc: true
authors: [chienkira]
---

**Qua cọ xát thực tế, mình mới ngộ ra được bản chất của cái limit về code storage trong AWS lambda. Bài này muốn chia sẻ lại một chút kiến thức có được với mọi người.**

## Bối cảnh
Công ty đang làm service chủ yếu theo kiến trúc serverless, nên dùng khá nhiều AWS Lambda functions.
- Số lượng Lambda: 138
- Code size của Lambda: average 14.4MB ~ max 50.7MB

## Diễn biến
Ngày đẹp trời nhận được thông báo CI failed.
Kiểm tra thì phát hiện ra là job deploy lambda đã không chạy thành công.
Log của serverless như sau:
```
Serverless: Operation failed!
 
   Serverless Error ---------------------------------------
    
      An error occurred: XXXXXXXXXXXXXXXXX - Code storage limit exceeded. (Service: AWSLambda; Status Code: 400; Error Code: CodeStorageExceededException; Request ID: XXXXXX).
       
```
Tưởng gì






