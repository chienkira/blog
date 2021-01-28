---
title: "Forever free servers with Oracle Cloud"
date: 2021-01-28T16:17:05+09:00
draft: false
tags: [cloud, infra]
language: vietnamese
toc: true
authors: [chienkira]
cover: /blog/images/free_oracle_cloud.png
---

**Như các bạn đã biết, các nền tảng IaaS phổ biến thì đều có chương trình trial hay free credits để cho người dùng dễ dàng tiếp cận cũng như trải nghiệm thử dịch vụ của họ (thu hút người dùng)
Thế nhưng cho sử dụng miễn phí mãi mãi thì hiện tại chỉ có Oracle cloud và Google cloud platform (theo như mình biết).
Tuy nhiên so sánh nội dung và phạm vi miễn phí thì anh Oracle cloud chơi lớn hơi hẳn! Cùng mình đăng ký sử dụng thử xem sao nhé.**

## What is free?

Trước tiên xem xem những gì được sử dụng miễn phí nhé.
Theo như quảng cáo trên trang chủ [tại đây](https://www.oracle.com/cloud/free/#always-free) thì hấp dẫn lắm luôn.

Mình tổng kết lại như thế này.

- **Compute VM:** 1/8 của Octa CPU tức là 1 CPU, 1 GB memory (gấp đôi so với GCP). Đã thế còn được miễn phí lên tới 2 servers! GCP chỉ được 1 thôi!
- **Block volume:** mỗi server được tối đa 50GB, x2 là 100GB (GCP hình như 30 hay 40GB gì thôi??)
- **Object storage:** miễn phí đến 10GB. Cái này giống kiểu S3 của AWS??
- **Database:** cho dùng database miễn phí nữa!! spec quảng cáo là 1 Octa CPU và 20 GB storage. Tuy nhiên hình như chỉ xài được Oracle DB??
- **Load balancer:** được miễn phí 1 instance nhé! băng thông 10 Mbps.
- Và 1 số dịch vụ để monitoring và notification nữa vân vân

Các bạn lưu ý Oracle cloud quảng cáo là **always free** nhé, không có giới hạn thời gian!
Có thể suy đoán rằng, vì launch ra thị trường muộn hơn các ông lớn khác, để đối phó với sự cạnh tranh khốc liệt họ buộc phải tung ra chương trình khuyến mãi hấp dẫn như thế này.
Mà nói chung người được lợi cuối cùng là chúng ta, dev quèn thích nghịch ngợm và không mất gì phải không ạ? :joy:

## Let's register

Đăng ký lấy 1 tài khoản Oracle cloud ở đây https://www.oracle.com/cloud/free/.

Mình thấy không có gì phức tạp và không nhiều bước đăng ký, chỉ cần làm theo các bước trên màn hình là dưới 10p đã có tài khoản rồi.

Chỉ có 2 lưu ý nhỏ thôi

- 1 là chỗ chọn Home Region, chọn region gần với bạn nhất vì always free chỉ áp dụng cho resource trong Home Region đã chọn thôi.
![](/blog/images/oracle_1.png)
- 2 là ngoài cần credit card, thì cần thêm 1 số điện thoại vì Oracle đòi xác nhận người thực qua SMS.

## Try creating our forever free server!

Có tài khoản rồi, giờ cùng mình thử tạo 1 server miễn phí nhé!

*Mình sử dụng home region là ap-tokyo-1 nên các đường link bên dưới sẽ có thể khác với các bạn.*

Vào phần [Compute / Instances](https://console.ap-tokyo-1.oraclecloud.com/compute/instances), chọn Create instance.

- Image mặc định là Oracle linux, mình đổi qua Canonical Ubuntu 20.04 Always Free Eligible
- Shape (là size của instance) thì để như mặc định là VM.Standard.E2.1.Micro Always Free Eligible
- Networking thì nhớ kiểm tra tick Assign a public IPv4 address để lấy IP nhé
- SSH keys tạo mới hoặc tốt nhất paste public key của bạn lên cho dễ
- Boot volume, cài đặt là 50 GB

![](/blog/images/oracle_3.png)

Nhấn Create và đợi instance khởi động xong. Thử ssh vào server nhé, do có spec và bandwidth cao hơn nên mình thấy tốc độ kết nối và phản hồi khi gõ lệnh cũng khác hẳn với cái instance khiêm tốn miễn phí của GCP :smiley:

![](/blog/images/oracle_4.png)

**Quan trọng**

Sau khi cài thử nginx, mình vẫn không thể access được web bằng public IP.
Mặc dù đã check kỹ lại setting của Security Lists và Security Groups nhưng vẫn không ăn thua.
Sau một hồi tìm hiểu thì mình xin tóm lược lại vấn đề và cách giải quyết ở đây cho các bạn đỡ mất thời gian google nhé.

Nguyên nhân là image Ubuntu trên Oracle cloud mặc định có 1 setting trong ip tables đang deny mọi request đến. Cái này mới nha, khác với AWS hay GCP nên các bạn cần chú ý nhé.

Cách giải quyết là xóa bỏ setting đó đi bằng lệnh iptables như bên dưới.

```
$ sudo iptables --list --line-numbers
# xác định line number của dòng DENY trong phần Chain INPUT rồi dùng lệnh sau để xóa dòng đó đi
# trường hợp của mình thì line number là 6 nên lệnh sẽ như sau:
$ sudo iptables -D INPUT 6
```

---

Server với spec như thế này dư sức hosting 1 trang web nhỏ hay dùng để chạy background tasks cho chúng ta. 
Ngoài ra Oracle cho phép sử dụng tới 2 server như vậy, miễn phí, và mãi mãi!! Mình thấy campaign hấp dẫn này vô tình cũng tạo ra một động lực nhỏ để chúng ta động tay động chân làm ra 1 cái gì đó ha, nên mình xin phép giới thiệu nó với các bạn qua blog này. Happy hacking!
