---
title: "Am Hiểu Blue Green Deployment Trong 5 Phút"
date: 2019-03-06T09:29:32+09:00
draft: true
tags: [aws, sre, infra]
language: vi
toc: true
authors: [chienkira]
---

**Blue green deployment (ở dưới mình sẽ gọi tắt là B/G deploy) là cái khỉ ho gì? Nó có gì hay và có "ngon" không?**

**Nếu bạn đang có câu hỏi tương tự trong đầu thì hãy thử đọc hết bài viết này nhé. Đây cũng là chia sẻ thực tế của mình sau khi được giao cho task thiết kế B/G deploy áp dụng lên hệ thống trong công ty.**

# Giới thiệu B/G deploy

Trước tiên cùng hình dung về infra của một hệ thống truyền thống. Trừ phục vụ cho môi trường dev hay stage ra, để cung cấp service cho users sử dụng - môi trường production, chúng ta thường sẽ sử dụng một nguồn tài nguyên phần cứng đúng không các bạn. Khi deploy một phiên bản mới, chúng ta deploy lên chính phần cứng đó - nơi service đang chạy.

Điều này dẫn đến một vấn đề là Downtime trong quá trình deploy, nói cách khác là users không thể sử dụng được service trong thời gian deploy. Nó lý giải tại sao người ta hay chọn deploy lúc nửa đêm, và tạo ra mấy màn hình thông báo Maintenance.

Để giải quyết vấn đề trên, B/G deploy được ra đời. Ý tưởng của nó là sử dụng **hai tài nguyên phần cứng song song** giống hệt nhau để tạo ra hai môi trường production (gọi là blue và green). Quá trình deploy bằng phương pháp B/G sẽ diễn ra như sau:

* Users sử dụng service thông qua 1 trong 2 môi trường - giả sử đang là Blue.
* Phiên bản mới sẽ được deploy lên môi trường còn lại - Green - môi trường mà users đang không sử dụng.
* Sau khi deploy hoàn tất và được kiểm tra xác định là ok thì ta chỉ việc route users qua môi trường Green.
* Môi trường Blue trở thành môi trường dự bị cho lần deploy tiếp theo

Các bạn thấy đó, vấn đề Downtime đã được giải quyết triệt để với B/G deploy.

Ở đây mình có lượm được từ google một hình vẽ khá là trực quan.

![blue-green-deploy](/static/images/bluegreen_deploy1.png)
*credit: https://www.blazemeter.com/blog/five-blue-green-deployment-best-practices-for-a-smooth-release*

### Ưu điểm

* Giảm thời gian downtime

  Như các bạn đã thấy ở phần giới thiệu phía trên, thời gian downtime do deployment đã được giải quyết. Users vẫn sử dụng service bình thường trong quá trình deploy, khi phiên bản mới sẵn sàng trên môi trường dự bị (green hay blue) thì request của users được route qua môi trường mới mà thôi. Quá trình deploy diễn ra mượt mà và không hề tạo ra bất kì phiền hà nào trong trải nghiệm với users.

* Dễ dàng rollback khi gặp trouble

  Bời vì chúng ta có sẵn hai môi trường production nên sau khi deploy, dù sự cố bất ngờ có phát sinh thì đường rút lui cũng luôn bật đèn xanh chờ sẵn. ;)

### "Thắt cổ chai" - nhược điểm

* Đắt đỏ
  
  Việc sử dụng đến hai môi trường production sẽ kéo theo vấn đề chi phí. Do đó quy mô service và ngân sách không hợp lý thì việc triển khai B/G deploy đôi khi biến thành "lợi bất cập hại".

* Deployment sẽ rất khó khăn khi có sự thay đổi lớn với database

  Phần database rất khó để cũng tách ra riêng biệt cho hai môi trường blue/green, do đó nó thường là phần giữ nguyên, không được switch khi deploy. Dẫn đến một việc là nếu trong lần release, schema của bảng dữ liệu có nhiều thay đổi thì deploy sẽ càng phức tạp. Có thể hình dung là đầu tiên ta phải thiết kế logic phía ứng dụng phiên bản mới sao cho tương thích với cả schema cũ, rồi deploy, rồi sau đó mới chỉnh sửa bỏ schema cũ để chỉ support với schema mới v...v...

# Chia sẻ thiết kế B/G deploy trong thực tế

Vào công ty, mình join vào team #SRE (Site Reliability Engineering) nên chẳng được tham gia vào phát triển product. Thay vào đó là làm mấy việc liên quan đến cải tiến quy trình nói chung, support team làm product, hay 1 số task pha chút devOps. Nói chung là chính mình cũng còn mơ hồ về công việc của team này :))

Thế rồi lúc vẫn chân ướt chân ráo, task đầu tiên mình được giao là áp dụng B/G deploy vào 1 số product của công ty. Từ đây mình mới đi tìm hiểu nó là cái vẹo gì rồi thiết kế và kiểm chứng mô hình có hoạt động hay không. Qua quá trình này, mình muốn chia sẻ những thông tin thực tế nhất mình hiểu được khi triển khai một B/G deploy.

### Bối cảnh quyết định áp dụng B/G deploy

Hệ thống của công ty mình thì toàn bộ nằm ở trên AWS, lại xây dựng theo kiến trúc serverless nên *pay as you go* trở thành một điểm cộng rất lớn khi triển khai B/G deploy. Bởi vì sao, vì chúng ta chỉ cần trả cho phần phát sinh sử dụng (tính theo số request, thời gian thực thi hàm lambda, lượng dữ liệu trung chuyển vân vân) chứ không phải trả khi tạo thêm tài nguyên. Do đó không chỉ blue và green, thậm chí tạo thêm red và brown cũng được. :))

Ngoài ra database sử dụng phần lớn là DynamoDB, đặc trưng của nó là schema không cố định, linh hoạt trên từng row (trừ thông tin key) nên vốn dĩ vụ release cũng không trở lên phức tạp khi có dependent database đi nữa.

### Quá trình thiết kế 
