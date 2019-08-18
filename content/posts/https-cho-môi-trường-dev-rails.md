---
title: "Https Cho Môi Trường Dev Rails"
date: 2019-08-18T21:47:30+09:00
draft: true
tags: [rails, HTTPS, ssl, docker]
language: vi
toc: false
authors: [chienkira]
---

**Hướng dẫn cấu hình tạo môi trường dev rails hỗ trợ giao thức HTTPS hoàn chỉnh nhanh và đơn giản nhất. Https "hiểu đơn giản kiểu đà điểu" là giao thức được secured, không trần truồng và lộ liễu như http thông thường.**

## Tại sao cần HTTPS cho môi trường dev?

Đúng rồi! Chắc sẽ có người thắc mắc vậy, "Ông dỗi hơi hay sao mà phải tạo môi trường dev hỗ trợ HTTPS, dev thì http là được rồi còn gì??"

Bình tĩnh nhé, để mình kể ra vài lý do cho.

- First rule, môi trường dev và môi trường production nên giống nhau hết sức có thể. Kể cả bạn chưa hiểu HTTPS với http khác nhau cái khỉ gì đi nữa, kệ bạn! Rule trên là rule trước hết bạn phải hiểu và tuân theo một khi bạn làm phát triển phần mềm. Thế các service hiện nay đang hoạt động trên giao thức nào là chính? Ngó qua báo cáo của Google ở đây [Percentage of HTTPS browsing time](HTTPS://transparencyreport.google.com/HTTPS/overview?hl=en) thử đi. Trên 90% và tiếp tục tăng lên là tỉ lệ của HTTPS browsing. Ở thời điểm này chắc bạn không muốn develop một thứ mà dự định sẽ chỉ chạy với giao thức http đâu.

- Rất nhiều third-party api bảo mật cao ví dụ liên quan đến Payment, hay Authentication đã cập nhật quy định, chỉ cho phép bạn sử dụng api của họ khi ứng dụng của bạn hoạt động trên giao thức HTTPS. Cho nên đôi khi không đơn thuần là good practise mà cuộc đời éo le ép bạn phải dev với HTTPS environment ấy! :joy:

- Các browser ngày nay đối xử khá "tệ nhạt" với http, ví dụ với trình duyệt phổ biến nhất hiện nay - Chrome, bạn sẽ gặp khá nhiều trouble khiến bạn mệt mỏi và mất thời gian nếu như môi trường dev của bạn không phải là HTTPS. Ví dụ mình hay gặp nhất là [mixed-content](https://developers.google.com/web/fundamentals/security/prevent-mixed-content/what-is-mixed-content).

- Vân vân mình nghĩ còn nhiều lắm, chưa gặp hết các vấn đề thôi.

## Yêu cầu đặt ra

Giờ cùng định nghĩ môi trường dev hỗ trợ HTTPS lý tưởng mà chúng ta cần là như thế nào nhé.

- Hỗ trợ HTTPS hoàn chỉnh. Hoàn chỉnh tức là không chơi kiểu HTTPS nhưng browser vẫn warning là invalid như thế này nhé.

    ![](https://user-images.githubusercontent.com/12954909/32688013-6b09c95e-c6d1-11e7-9501-aa952f232bbc.png)

    Sau một thời gian thực tế có dùng kiểu tạm bợ thế này, mình nhận ra khá nhiều vấn đề không ổn. Nhức nhối nhất là vụ Chrome sẽ coi trang web là Not secure, bỏ qua không cache requests và hiện cảnh báo "Form submission" rất khó chịu mỗi lần back - previous.

    Hỗ trợ giao thức HTTPS hoàn chỉnh, mình muốn nó phải kiểu như sau.

    ![](https://cdn-media-1.freecodecamp.org/images/1*89r7TnYG49V3zMoUnfOP7Q.png)

- Môi trường dev được sử dụng bởi nhiều thành viên trong team, vì vậy tính năng HTTPS này nên được áp dụng một cách dễ dàng. Tốt nhất là "ship" 1 phát ăn liền, mọi người đồng loạt được update từ http lên HTTPS! Chúng ta không muốn phải cài đặt cấu hình này nọ phức tạp cho từng máy từng người để hỗ trợ HTTPS cho cả team phải không?!

## Thực hiện thôi

- 