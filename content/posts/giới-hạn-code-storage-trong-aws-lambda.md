---
title: "Giới Hạn Code Storage Trong Aws Lambda"
date: 2019-04-15T17:37:58+09:00
draft: no
tags: [til, aws, lambda, serverless]
language: vi
toc: false
authors: [chienkira]
cover: /blog/images/lambda_dashboard_limits_cover.png
---

**Cọ xát thực tế mới ngộ ra được bản chất của cái limit code storage trong AWS lambda. Bài này muốn chia sẻ lại một chút kiến thức biết được với mọi người.**

## Bối cảnh
Công ty đang làm service chủ yếu theo kiến trúc serverless, nên dùng tương đối nhiều AWS Lambda functions.
- Số lượng Lambda: 238
- Code size của Lambda: average 20MB ~ max 50MB

## Diễn biến
Ngày đẹp trời nhận được thông báo **CI job failed**.
Kiểm tra thì phát hiện ra job **deploy hàm lambda** đã không chạy được thành công.

Log của serverless như sau:
```
Serverless: Operation failed!
 
   Serverless Error ---------------------------------------
    
      An error occurred: XXXXXXXXXXXXXXXXX - Code storage limit exceeded. (Service: AWSLambda; Status Code: 400; Error Code: CodeStorageExceededException; Request ID: XXXXXX).
       
```
Tưởng gì `Code storage limit exceeded` thì chắc là có function nào bự quá quy định rồi.
Một function Lambda thì chỉ được tối đa 50 MB zipped thôi, chắc có anh nào nhồi nhét tạo ra một cái hàm bự chà bá chứ gì?

Nghĩ vậy, mình dòm lên những dòng log phía trên tìm manh mối.
```
Serverless: Uploading service .zip file to S3 (14.4 MB)...
```
Vậy là đếch phải rồi, code size của function Lambda hoàn toàn bình thường như cân đường hộp sữa!

Đến lúc phải nhờ sensei Google chỉ giáo, sensei bảo rằng ngoài cái giới hạn ở trên, còn có một cái giới hạn khác mà AWS đặt ra áp dụng lên từng region của từng tài khoản.

Giới hạn đó như sau:

| Resource                                        | Default Limit |
| :---------------------------------------------- | :------------ |
| Concurrent executions                           | 1,000         |
| Function and layer storage                      | 75 GB         |
Ref: [AWS Lambda Limits](https://docs.aws.amazon.com/lambda/latest/dg/limits.html)

Code storage tối đa cho phép là 75GB cho tổng số toàn bộ Lambda functions trên 1 region.

Ta có thể check xem hiện tại ta đang xài đến bao nhiêu trong 75GB bằng cách mở màn hình [Dashboard Lambda](https://ap-northeast-1.console.aws.amazon.com/lambda/home?region=ap-northeast-1#/) và kiểm tra phần thông tin hiển thị phía trên đầu trang.

Lúc xảy ra lỗi này thì Code storage của mình đã là 100%.  
*Ảnh minh họa sau mình chụp sau khi xử lý rồi.*
![](/blog/images/lambda_dashboard_limits.png)

Vậy là biết thêm một cái limit mới, cái này nếu không dùng quá nhiều Lambda thì chắc tới tết mới chạm được vạch 75GB :)) Mà nghĩ lại thì việc công ty đã dùng đến 75GB vẫn có gì đó "ảo tung chảo" quá.
Nhẩm thử, cứ cho là 1 Lambda có 50MB code size đi chăng nữa thì tổng lại vẫn chỉ là như sau thôi.
```
50MB * 238funtions / 1024 = 11.6GB
```

**Thế tại sao lại bị AWS tính là 100% của 75GB rồi?!?!**

Đây mới là cái mình muốn chia sẻ với mọi người nhất.
Như mọi người đã biết, Lambda function có thể lưu lại các versions.
Mấu chốt ở đây là ta đã quên đi **phần dung lượng cần thiết để lưu các version cũ**.

Ở trường hợp của mình, mỗi lần serverless deploy thì đều tạo ra 1 version mới. Do đó mà code storage cứ tăng vèo vèo không thương tiếc mỗi lần CI chạy deploy. CI tiện lợi càng "thúc đẩy" các anh chị em dev và push push tới tấp, thế thì 75GB chứ 750GB cũng không trụ được lâu ấy chứ :))

## Xử lý
Có 2 hướng xử lý có thể xem xét đến.

- Nếu như ta cần thiết phải giữ versioning ở trên aws lambda, ta có thể dùng đến plugin [serverless-prune-plugin](https://www.npmjs.com/package/serverless-prune-plugin). Hàng này khá xịn, nó có thể tự động giữ đến n version mới nhất trên aws, các version cũ hơn sẽ được kiểm tra và xóa - prune khỏi aws mỗi lần deploy.

- Còn nếu như ta xác định khỏi cần thiết giữ các version cũ của lambda trên aws, giữ trên git là đủ :)) thì ta có thể lựa chọn disable versioning bằng setting sau.

   ```yaml
   provider:
   ...
   versionFunctions: false # optional, default is true
   ...
   ```

Và nhớ trước khi xử lý gì thì cũng nên làm 1 cái script để xóa vợi các version không cần thiết cho code storage giảm xuống dưới 100% đã nhé ạ. Không thì chả deploy nổi lên nữa để mà xử lý đâu mọi người ạ. :smiley:
