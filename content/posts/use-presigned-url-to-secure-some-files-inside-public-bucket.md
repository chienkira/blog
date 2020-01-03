---
title: "Use Presigned Url to Secure Only Some Files Inside A Public S3 Bucket"
date: 2020-01-03T21:17:52+09:00
draft: true
tags: []
language: vi
toc: true
authors: [chienkira]
cover: /blog/images/s3-bucket.png
---

**Tết mải ăn quá đến cuối ngày mồng 3 mới mò lên khai phím được :smiley:. Để chào mừng năm 2020 rực rỡ :fireworks:, bài viết lần này mình muốn note lại chia sẻ cách đặt policy cho S3 bucket, để vừa public bucket lại vẫn bảo mật (yêu cầu presigned url để truy cập) cho một số file nhất định trong bucket nha! Ok let's go!**

## Presigned URL là?

Nó là... như cái tên `Presigned URL` của nó thôi :)) Các bạn làm IT thì dù chưa biết nhưng nghe cái tên chắc hiểu đại khái nó là gì luôn rồi ạ. Trích dẫn từ tài liệu của aws như sau:

> the object owner can optionally share objects with others by creating a presigned URL, using their own security credentials, to grant time-limited permission to download the objects.

Ref: [docs.aws.amazon.com/AmazonS3/latest/dev/ShareObjectPreSignedURL](https://docs.aws.amazon.com/AmazonS3/latest/dev/ShareObjectPreSignedURL.html)

Với 1 URL bình thường kiểu `https://foobar-bucket.aws/a_hot_image.png`, sẽ hoạt động với chỉ một điều kiện duy nhất - object a_hot_image.png là public. Nghĩa là chỉ cần biết url thì ai cũng có thể truy cập được nội dung bên trong. Trong thế giới web điều này được coi là hiển nhiên, vì web sinh ra là để public nội dung cho nào là browser nào là api client vân vân mây mây... truy cập vào mà. Tuy nhiên không cẩn thận bạn có thể bị dính vào những vấn đề security nghiêm trọng nếu như nội dung các object đó là loại "nhạy cảm" - sentive data đấy!

Ví dụ, hệ thống của bạn có chức năng cho phép người dùng đăng ảnh xác thực cá nhân (ảnh bằng lái xe chẳng hạn) lên, nếu bạn không secure những dữ liệu ấy, bên thứ 3 bằng cách có được url (qua đánh cắp log trình duyệt, qua nghe lén network...) sẽ có thể xem được những dữ liệu vô cùng là riêng tư đó :scream:

Rất may AWS S3 cung cấp cho chúng ta tính năng Presigned URL giống như một giải pháp để chúng ta chủ động hạn chế truy cập đến các nội dung sensitive. URL có chữ ký - Presigned URL khác với URL thông thường ở chỗ nó được gắn thêm một tá GET params hoành tráng. Kiểu như sau: 
`https://foobar-bucket.aws/a_hot_image.png?X-Amz-Algorithm=xxxxxxxxxx&X-Amz-Credential=xxxxxxxxxx&X-Amz-Date=xxxxxxxxxx&X-Amz-Expires=3600&X-Amz-SignedHeaders=host&X-Amz-Signature=xxxxxxxxxx`
Các tham số truyền vào này chỉ AWS mới decode và validate được (dựa vào IAM của bạn sử dụng để sinh ra presigned url), chỉ khi là valid và còn trong thời hạn hiệu lực thì AWS mới trả về nội dung của object. Nôm na là chữ ký phải đúng và mực chưa bay màu thì mới có xiền á :smiley:.

Lưu ý hạn expire của presigned url dài nhất mà aws cho phép cài đặt là 7 days! Nhưng về cơ bản, mình khuyến khích mọi người nên để ngắn hơn - từ vài phút đến vài tiếng để giữ an toàn tối đa cho url.

## Yêu cầu presigned url cho một số file nhất định mà thôi?!

### Tại sao có tình huống như thế?

Càng ngày việc dùng S3 để làm storage cho hệ thống web càng phổ biến. Vì nó rẻ, nhanh, đỡ 1 phần load cho web server của bạn, bạn cũng không phải lo quản lý storage size...
Dữ liệu trong hệ thống web thì chia ra làm hai loại.
- Loại public được và cần phải public, ví dụ như ảnh sản phẩm, ảnh banner, logo vân vân, web để thiên hạ xem mà có phải giấu kín đâu phải không ạ.
- Loại sensitive, cần bảo mật. Như ví dụ ở trên mình đã đưa ra, các dữ liệu mà người dùng thành viên upload lên, đặc biệt là mấy thứ có liên quan đến thông tin cá nhân hay thanh toán!

Để đáp ứng thỏa mãn yêu cầu lưu trữ cho 2 loại dữ liệu trên, cách chân phương nhất có lẽ là sử dụng 2 buckets. Một bucket để public và bucket còn lại để private, truy cập object thông qua presigned url. Cách làm này không có vấn đề gì nhưng theo mình trong quá trình vận hành sẽ gặp đôi chút khó khăn và không "clean" lắm vì mỗi môi trường lúc nào cũng cần tới 2 buckets.

Do đó mình đã nghĩ chả lẽ không có cách giải quyết sao cho chỉ sử dụng một S3 bucket sao? Lần này không biết do google ngu hay sao mà google không ra luôn... cuối cùng mình đã phải tự mò mẫm tìm cách làm. Vì đã thành công rồi nên muốn note lại đây để chia sẻ với các bạn.

### Cài đặt policy để giải quyết vấn đề

Đầu tiên, ta phải public mọi object trong bucket để thỏa mãn yêu cầu thứ nhất - lưu trữ các dữ liệu public.
Policy để làm việc đó thì đơn giản như sau.

```xml
{
    "Version": "2012-10-17",
    "Id": "bucket_policy",
    "Statement": [
        {
            "Sid": "public_all_files",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::app-files-prod/*"
        }
    ]
}
```

Với policy này, statement `public_all_files` sẽ đảm bảo mọi object trong bucket `app-files-prod` có thể được truy cập bằng URL bình thường ví dụ như https://app-files-prod.s3.ap-northeast-1.amazonaws.com/xxxxxxx.png.

Tiếp theo giả sử ta cần bảo mật các files trong thư mục identity_images và personal_images sao cho chúng chỉ truy cập được thông qua presigned url, bạn hãy thêm 1 statement vào tiếp policy như dưới đây.

```xml
{
    "Version": "2012-10-17",
    "Id": "bucket_policy",
    "Statement": [
        {
            "Sid": "public_all_files",
            ...
        },
        {
            "Sid": "secure_sensitive_files",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": [
                "arn:aws:s3:::app-files-prod/*/identity_images/*",
                "arn:aws:s3:::app-files-prod/*/personal_images/*"
            ],
            "Condition": {
                "StringNotEquals": {
                    "s3:signatureversion": "AWS4-HMAC-SHA256"
                }
            }
        }
    ]
}
```

Giải thích ra thì statement mới thêm `secure_sensitive_files` làm nhiệm vụ `Deny` hành động `s3:GetObject` cho các objects trong 2 folders trên nếu như AWS không validate được là presigned url version `AWS4-HMAC-SHA256` hợp lệ (presigned url version 4 - mới nhất hiện nay)

Sau khi áp dụng policy, các objects trong thư mục identity_images hay personal_images khi truy cập với URL bình thường sẽ bị trả về lỗi. Chỉ khi access bằng presigned url thì nội dung ảnh mới được download về. Còn các objects còn lại trong bucket vẫn được public như bình thường! 
![](/blog/images/s3_access_deny.png)

23:51 rồi, suýt chút nữa là hết ngày mồng 3 - hết tết. Bài viết này kết thúc ở đây để nó còn kịp là bài viết khai phím tết 2020 :))
Chúc mọi người và xin phép chúc chính mình một năm mới nhiều gặt hái, nhiều niềm vui và may mắn nha!