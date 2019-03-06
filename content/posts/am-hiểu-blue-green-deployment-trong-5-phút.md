---
title: "Am Hiểu Blue Green Deployment Trong 5 Phút"
date: 2019-03-06T09:29:32+09:00
draft: true
tags: [aws, sre, infra]
language: vi
toc: true
authors: [chienkira]
---

**Blue green deployment là cái khỉ ho gì? Nó có gì hay và có "ngon" không?**
**Nếu bạn đang có câu hỏi tương tự trong đầu thì hãy thử đọc hết bài viết này nhé. Đây cũng là chia sẻ thực tế của mình sau khi được giao cho task thiết kế Blue green deployment áp dụng lên hệ thống trong công ty.**

# Giới thiệu Blue Green deployment
  *từ giờ viết gọn là B/G deploy*

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

## Ưu điểm

* Giảm thời gian downtime

  Như các bạn đã thấy ở phần giới thiệu phía trên, thời gian downtime do deployment đã được giải quyết. Users vẫn sử dụng service bình thường trong quá trình deploy, khi phiên bản mới sẵn sàng trên môi trường dự bị (green hay blue) thì request của users được route qua môi trường mới mà thôi. Quá trình deploy diễn ra mượt mà và không hề tạo ra bất kì phiền hà nào trong trải nghiệm với users.

* Dễ dàng rollback khi gặp trouble

  Bời vì chúng ta có sẵn hai môi trường production nên sau khi deploy, dù sự cố bất ngờ có phát sinh thì đường rút lui cũng luôn bật đèn xanh chờ sẵn. ;)

## "Thắt cổ chai" - nhược điểm

* Đắt đỏ
  
  Việc sử dụng đến hai môi trường production sẽ kéo theo vấn đề chi phí. Do đó quy mô service và ngân sách không hợp lý thì việc triển khai B/G deploy đôi khi biến thành "lợi bất cập hại".

* Deployment sẽ rất khó khăn khi có sự thay đổi lớn với database

  Phần database rất khó để cũng tách ra riêng biệt cho hai môi trường blue/green, do đó nó thường là phần giữ nguyên, không được switch khi deploy. Dẫn đến một việc là nếu trong lần release, schema của bảng dữ liệu có nhiều thay đổi thì deploy sẽ càng phức tạp. Có thể hình dung là đầu tiên ta phải thiết kế logic phía ứng dụng phiên bản mới sao cho tương thích với cả schema cũ, rồi deploy, rồi sau đó mới chỉnh sửa bỏ schema cũ để chỉ support với schema mới v...v...

# Chia sẻ thiết kế B/G deploy trong thực tế

Vào công ty, mình join vào team #SRE (Site Reliability Engineering) nên chẳng được tham gia vào phát triển product. Thay vào đó là làm mấy việc liên quan đến cải tiến quy trình nói chung, support team làm product, hay 1 số task pha chút devOps. Nói chung là chính mình cũng còn mơ hồ về công việc của team này :))

Thế rồi lúc vẫn chân ướt chân ráo, task đầu tiên mình được giao là áp dụng B/G deploy vào 1 số product của công ty. Từ đây mình mới đi tìm hiểu nó là cái vẹo gì rồi thiết kế và kiểm chứng mô hình có hoạt động hay không. Qua quá trình này, mình muốn chia sẻ những thông tin thực tế nhất mình hiểu được khi triển khai một B/G deploy.

## Bối cảnh quyết định áp dụng B/G deploy

Hệ thống của công ty mình thì toàn bộ nằm ở trên AWS, lại xây dựng theo kiến trúc serverless nên *pay as you go* trở thành một điểm cộng rất lớn khi triển khai B/G deploy. Bởi vì sao, vì chúng ta chỉ cần trả cho phần phát sinh sử dụng (tính theo số request, thời gian thực thi hàm lambda, lượng dữ liệu trung chuyển vân vân) chứ không phải trả khi tạo thêm tài nguyên. Do đó không chỉ blue và green, thậm chí tạo thêm red và brown cũng được. :))

Ngoài ra database sử dụng phần lớn là DynamoDB, đặc trưng của nó là schema không cố định, linh hoạt trên từng row (trừ thông tin key) nên vốn dĩ vụ release cũng không trở lên phức tạp nhiều khi có dependent database đi nữa.

## Quá trình thiết kế 

### Bản nháp ý tưởng

Mấu chốt của B/G deploy là làm sao xây dựng ra cơ chế cho phép ta tùy ý route (chuyển) các requests từ users tới 1 trong 2 môi trường Blue và Green.

Sau khi đào bới thông tin trên internet một hồi, mình nhận ra rằng cách phổ biến để xử lý việc routing request này là thông qua DNS. Cụ thể là với Route53 của AWS, ta có thể thực hiện cơ chế trên như sau:
- giả sử domain truy cập service là https://example.com , và ta có 2 CloudFront ứng với 2 môi trường Blue Green có domain tương ứng là https://d000blue.cloudfront.net và https://d000green.cloudfront.net
- ta cài đặt 2 weighted DNS CNAME record trỏ đến 2 CloudFront trên
- khi cần route đến môi trường blue, ta điều chỉnh `weight` của 2 DNS record trên thành `weight: 100` ứng với https://d000blue.cloudfront.net và `weight: 0` ứng với domain còn lại
- ngược lại khi cần đổi lại môi trường green, ta lại điều chỉnh giá trị `weight` thành `100` cho record trỏ đến https://d000green.cloudfront.net là xong

Cách giải quyết này khá là dễ hiểu và thực hiện. Tuy nhiên cũng có một điểm hơi khiến mình băn khoăn đó là, lợi dụng setting của DNS thì sẽ bị phụ thuộc vào spec của DNS server. Cụ thể hơn, mình băn khoăn ở chỗ, thời gian cần thiết để thay đổi DNS setting có hiệu lực với toàn bộ người dùng ở đây là không kiểm soát được. Mình không thích "mất kiểm soát" như vậy. :))

Đi tìm cách khác để thực hiện cơ chế routing này, mình nhớ đến ứng viên mà mình đã thấy rất tiềm năng khi tìm hiểu về CloudFront - **Lambda@Edge**. CloudFront cho phép trigger Lambda@Edge mỗi khi nó request nội dung từ origin để phục vụ users. Ý tưởng ở đây sẽ là thực hiện việc routing ở trong Lambda@Edge, nghĩa là ta sẽ điều hướng CloudFront lấy content từ origin mà ta muốn (origin của môi trường Blue hoặc Green). Flag để xác định môi trường Blue hay Green đang active thì mình dùng 1 biến lưu trong Parameter store, vừa dễ tham chiếu lại vừa dễ thay đổi ;)

Thiết kế cuối cùng mà mình nghĩ ra như trong bản nháp dưới đây.

Trong lúc vẽ nháp, mình còn ngộ ra một chỗ rất "ăn điểm" trong thiết kế này. Đó là flag lưu trong Parameter store có thể dùng làm luôn định vị để cấu hình cho CircleCI tự động biết deploy lên môi trường không active. Tự động hóa hết rồi, vậy là chỉ việc dev và dev đến chết, lúc nào cần switch môi trường để release thì cập nhật cái flag là ok thôi.

![blue-green-prototype](/static/images/bluegreen_deploy_draft1.jpg_)

### Thiết kế cuối cùng

Lúc đầu mình định đưa lên đây ảnh architecture đẹp đẽ đã vẽ, nhưng nghĩ lại đó rốt cục là cũng là tài liệu thiết kế trong công ty nên không public thì tốt hơn.

Cơ bản thì thiết kế như bản nháp trên mình đã giới thiệu, mình chỉ đưa thêm một feature nhỏ dạng backdoor vào để team dev product dễ dàng tùy ý kiểm thử môi trường blue/green hơn thôi. Các bạn cũng biết đó, không check thử tí nào mà switch môi trường production thì khá là mạo hiểm mà.

# Thông tin bên lề biết được thêm khi tìm hiểu về B/G deploy

* A/B testing
* Canary test
