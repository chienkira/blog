---
title: "Https Cho Môi Trường Dev Rails"
date: 2019-08-18T21:47:30+09:00
draft: no
tags: [rails, HTTPS, SSL, docker]
language: vi
toc: false
authors: [chienkira]
cover: /blog/images/https_done.png
---

**Hướng dẫn cấu hình tạo môi trường dev rails hỗ trợ giao thức HTTPS hoàn chỉnh nhanh và đơn giản nhất. Https "hiểu đơn giản kiểu đà điểu" là giao thức được secured, không trần truồng và lộ liễu như http thông thường.**

## Tại sao cần HTTPS cho môi trường dev?

Đúng rồi! Chắc sẽ có người thắc mắc vậy, "Ông dỗi hơi hay sao mà phải tạo môi trường dev hỗ trợ HTTPS, dev thì http là được rồi còn gì??"

Bình tĩnh nhé, để mình kể ra vài lý do cho.

- First rule, môi trường dev và môi trường production nên giống nhau hết sức có thể. Kể cả bạn chưa hiểu HTTPS với http khác nhau cái khỉ gì đi nữa, kệ bạn! Rule trên là rule trước hết bạn phải hiểu và tuân theo một khi bạn làm phát triển phần mềm. Thế các service hiện nay đang hoạt động trên giao thức nào là chính? Ngó qua báo cáo của Google ở đây [Percentage of HTTPS browsing time](HTTPS://transparencyreport.google.com/HTTPS/overview?hl=en) thử đi. Trên 90% và tiếp tục tăng lên là tỉ lệ của HTTPS browsing. Ở thời điểm này chắc bạn không muốn develop một thứ mà dự định sẽ chỉ chạy với giao thức http đâu.

- Rất nhiều third-party api bảo mật cao ví dụ liên quan đến Payment, hay Authentication đã cập nhật quy định, chỉ cho phép bạn sử dụng api của họ khi ứng dụng của bạn hoạt động trên giao thức HTTPS. Cho nên đôi khi không đơn thuần là good practise mà cuộc đời éo le ép bạn phải dev với HTTPS environment ấy! :joy:

- Các browser ngày nay đối xử khá "tệ nhạt" với http, ví dụ với trình duyệt phổ biến nhất hiện nay - Chrome, bạn sẽ gặp khá nhiều trouble khiến bạn mệt mỏi và mất thời gian nếu như môi trường dev của bạn không phải là HTTPS. Điền hình mình thấy mọi người hay gặp nhất là vụ [mixed-content](https://developers.google.com/web/fundamentals/security/prevent-mixed-content/what-is-mixed-content).

- Vân vân mình nghĩ còn nhiều lắm, chưa gặp hết các vấn đề thôi.

## Yêu cầu đặt ra

Giờ cùng định nghĩ môi trường dev hỗ trợ HTTPS lý tưởng mà chúng ta cần là như thế nào nhé.

- Hỗ trợ HTTPS hoàn chỉnh. Hoàn chỉnh tức là không chơi kiểu HTTPS nhưng browser vẫn warning là invalid như thế này nhé.

    ![](https://user-images.githubusercontent.com/12954909/32688013-6b09c95e-c6d1-11e7-9501-aa952f232bbc.png)

    Sau một thời gian thực tế dùng kiểu tạm bợ thế này, mình nhận ra có khá nhiều vấn đề không ổn. Nhức nhối nhất là vụ Chrome sẽ coi trang web là Not secure, bỏ qua không cache requests và hiện cảnh báo "Form submission" rất khó chịu mỗi lần back - previous.

    Hỗ trợ giao thức HTTPS hoàn chỉnh thì nó phải kiểu như sau.

    ![](https://cdn-media-1.freecodecamp.org/images/1*89r7TnYG49V3zMoUnfOP7Q.png)

- Môi trường dev được sử dụng bởi nhiều thành viên trong team, vì vậy tính năng HTTPS này nên được áp dụng một cách dễ dàng. Tốt nhất là "ship" 1 phát ăn liền, mọi người đồng loạt được update từ http lên HTTPS! Chúng ta không muốn phải cài đặt cấu hình này nọ phức tạp cho từng máy từng người để hỗ trợ HTTPS cho cả team phải không?!

## Thực hiện

Sau đây, mình muốn giới thiệu combo mà mình thấy đơn giản và nhanh gọn nhất, đảm bảo trong 10 phút bạn có thể cài đặt được hoàn chỉnh HTTPS! 

*Hướng dẫn cụ thể bên dưới là cho Rails, nhưng các framework khác mình nghĩ cũng áp dụng tương tự được.*

1. Bước 1: Tạo SSL certificate

    - Giao thức HTTPS cần gì đầu tiên? chính là cần một **SSL certificate** hợp lệ, được ký và xác thực bởi một **Certificate Authorities** gọi tắt là CA đủ tin tưởng. Nếu như không đủ những điều kiện trên, trình duyệt sẽ báo lỗi "Not Secure" mình nói ở trên.

    - Vì ở đây chúng ta đang xử lý cho môi trường dev, không phải production nên SSL cert sẽ là self-signed SSL cert và CA cũng là CA "dỏm" ta tự lập ra.

    - Bạn có thể đầu tư thời gian Google tìm hiểu và tự tạo SSL cert, rồi cách cài đặt để máy trust cái CA "dỏm" đã tạo ra. Nhưng mất thời gian lắm, làm thế hết xừ 10 phút rồi :joy: Xin giới thiệu và chào mời bạn dùng tool [https://github.com/FiloSottile/mkcert](https://github.com/FiloSottile/mkcert) nhé.

        > A simple zero-config tool to make locally trusted development certificates with any names you'd like.

    - Bắt tay vào làm như bên dưới. Sau khi tạo xong bạn có thể mở Keychain lên kiểm tra Cert đã được import và trust thành công như hình dưới.
        ```bash
        $ brew install mkcert  # cài đặt mkcert
        $ mkcert -install      # tạo CA
        $ mkcert localhost     # tạo SSL cert cho domain localhost
        ```
    ![](/blog/images/https_keychain.png)
    

1. Bước 2: Cấu hình Rails/Puma hoạt động với SSL cert đã tạo ở bước 1

    - Khi tạo SSL cert bằng lệnh `mkcert localhost` ở trên, để ý output ta sẽ thấy 2 files gồm 1 file là cert file và 1 file là key đã được tạo ra.

        ```bash
            $ mkcert localhost
            Using the local CA at "/Users/kira/Library/Application Support/mkcert" ✨

            Created a new certificate valid for the following names 📜
            - "localhost"

            The certificate is at "./localhost.pem" and the key at "./localhost-key.pem" ✅
        ```

    - Tạo 1 thư mục ví dụ `config/certs` rồi copy 2 files trên. *Đây là cert cho domain localhost - môi trường dev, nên không cần đưa vào .gitignore. Nhưng nếu là file cert xịn thì đừng quên bỏ nó ra khỏi git repository nhé!*
    ```bash
    $ cp localhost* /path/to/your_rails_app/config/certs/
    ```

    - Mở file `config/puma.rb` thêm vào đoạn code sau để cấu hình puma khởi động web server với SSL cert bên trên. Ở đây để linh hoạt mình sẽ để puma khởi động với giao thức http thường nếu như kiểm tra không có files trong folder certs.

        ```ruby
        if Rails.env.development?
            key_file = Rails.root.join("config", "certs", "localhost-key.pem")
            cert_file = Rails.root.join("config", "certs", "localhost.pem")

            if key_file.exist?
                ssl_bind "0.0.0.0", "3000", {
                    key: key_file.to_path,
                    cert: cert_file.to_path,
                    verify_mode: 'none'
                }
            else
                bind "tcp://0.0.0.0:3000"
            end
        end
        ```
1. Bước 3: Thực ra là xong rồi, chỉ kiểm tra lại để thấy rails app đã chạy với HTTPS thôi.
    ```bash
    $ bin/rails s   # khởi động dev server
    ```

    Mở trình duyệt và truy cập thử https://localhost:3000 . Thành công rồi ta sẽ thấy giống hình bên dưới. Nếu bạn nhanh, mình nghĩ còn không tốn đến 5 phút!

    ![](/blog/images/https_done.png)

1. Bước 4: Đến đây thì thực ra mọi thứ mới chỉ hoạt động tốt trên máy của bạn. Nếu bạn thử truy cập từ máy khác, lỗi "Not Secure" sẽ lại hiện lên. Vì sao ư? Vì hiện tại ta đang xài đồ self-signed, không phải là một cert valid được xác thực bởi một CA thực sự. Do đó chỉ những thiết bị nào được cài đặt trust cert trên thì mới không báo lỗi "Not Secure".

    Vậy thì để các máy của các thành viên khác trong team chịu "nhận" self-signed này, bạn cần bảo mọi người chạy command sau. *Chỉ 1 command thôi, là đâu vào đấy*

    ```bash
    $ sudo security add-trusted-cert -d -r trustAsRoot -k /Library/Keychains/System.keychain /path/to/your_rails_app/config/certs/localhost.pem
    ```

---

Đã một thời gian hơn 1 tháng mình không update blog, một chút thay đổi trong cuộc sống và công việc công ty làm mình bị khá bận và mệt. Thời gian tới dự là học được nhiều cái hay ho nữa, nếu có thời gian nhất định sẽ update tiếp blog!