---
title: "Better Logging Practice (Part 1)"
date: 2020-05-24T13:17:39+09:00
draft: false
tags: [rails, logging, tech]
language: vi
toc: true
authors: [chienkira]
cover: /blog/images/Better_logging_practice.png
---

**Ruby là một ngôn ngữ đẹp, gems lại giống những món trang sức vừa xứng tầm vừa sang chảnh của ả.**
**Và cá nhân mình cảm nhận thấy gu của những dev làm việc với Ruby cũng ít nhiều ở một level khá ổn, mọi người khá "sạch sẽ" và "tinh tế" :)) ngay cả trong chuyện logging nhé! Qua bài này, xin giới thiệu 1 gem và 1 AWS tool hữu ích cho việc quản lý logs đến các thánh thần! 2 3 Zô :beer:!**

## Khoan, sao cần better logging pratice?

Các cụ ngày trước cứ dạy là *tốt gỗ hơn tốt nước sơn* đúng không?

Nước sơn thì giống UI của hệ thống, còn gỗ tốt hay không thì cái vụ logging này nó cũng quyết định lắm nè!
Nếu không coi trọng logging, khi hệ thống bị bug hay bị attack (gọi là gỗ bị sâu) 
thì, hoặc là chả coi được log hoặc là log có đó mà không xài được dẫn đến hoàn cảnh bế tắc, 
lâu dần càng ngày gỗ càng mục nát mà! Cuối cùng người đau khổ và phải hấng chịu nhiều nhất vẫn chính là các bạn - người làm ra hệ thống. 

※ Ở đây xin phép chỉ đề cập đến better chứ không phải best vì mình chả phải là một chuyên gia Devops.
Tuy nhiên với những hệ thống quy mô trung bình, được thiết kế scale ngang nhiều server chẳng hạn đi nữa, mình nghĩ pratice này vẫn dư sức đáp ứng được. Vì thế rất recommend mọi người cài đặt và cải thiện hệ thống để nó ngày càng "tốt gỗ" hơn nhé.

## Giới thiệu gem lograge

###### Link Github https://github.com/roidrage/lograge
> Lograge is an attempt to bring sanity to Rails' noisy and unusable, unparsable and, in the context of running multiple processes and servers, unreadable default logging output. Rails' default approach to log everything is great during development, it's terrible when running it in production. It pretty much renders Rails logs useless to me.

###### Lograge cho bạn những dòng log có tổ chức và parsable!

Ai đã làm việc với Rails thì không 1 lần cũng là vô số lần thấy những dòng log mặc định như bên dưới nhỉ.

```bash
Started GET "/" for 127.0.0.1 at 2012-03-10 14:28:14 +0100
Processing by HomeController#index as HTML
  Rendered text template within layouts/application (0.0ms)
  Rendered layouts/_assets.html.erb (2.0ms)
  Rendered layouts/_top.html.erb (2.6ms)
  Rendered layouts/_about.html.erb (0.3ms)
  Rendered layouts/_google_analytics.html.erb (0.4ms)
Completed 200 OK in 79ms (Views: 78.8ms | ActiveRecord: 0.0ms)
```

Rất nhiều thông tin chi tiết, rất hữu ích cho... **lúc đang development** thôi các thánh thần ạ!
Trong môi trường production, log mặc định này ngược lại trở thành cồng kềnh và đồ xộ không cần thiết.
Ngoài ra như bạn đã nhận ra, nội dung log không có cấu trúc nên không phải parsable. Thế thì làm sao bạn phân tích được log bằng tool hay script do bạn viết? 

Chưa kể thử hình dung hệ thống có nhiều process cùng đồng thời log theo format trên, các dòng log sẽ bị mix vào nhau :scream: và thế là... không có cơ hội nào cho bạn xử lý file log đó nữa. Giống như mụ Gì Ghẻ trộn gạo và đỗ xanh rồi bắt cô Tấm nhặt đằng nào ra đằng nấy ấy, tiếc là hông có Bụt cứu ta đâu :))

Vậy thì Lograge đề xuất lên 1 format log mới, cô đọng hơn và quan trọng nhất là parsable.
Nó hỗ trợ vài format log khác nhau, dưới đây sample 1 dòng log với format mặc định của Lograge.

```bash
method=GET path=/jobs/833552.json format=json controller=JobsController  action=show status=200 duration=58.33 view=40.43 db=15.26
```

Log đã được tóm lược lại trên 1 dòng, định dạng key=value và được rút gọn bỏ đi rất nhiều thông tin rườm rà trước đó.
Vì cũng chưa có timestamp nên trong thực tế khi sử dụng lograge người ta thường chọn 1 format khác - JSON và thêm nhiều thông tin log cần thiết khác.
Sau đây hướng dẫn các bạn cách cài đặt và giới thiệu cấu hình lograge mình từng sử dụng cho các bạn tham khảo.

### Hướng dẫn cài đặt Lograge

1. Cài đặt lograge
    
Thêm vào Gemfile và chạy lại bundle install

```ruby
gem "lograge"
```

2. Cấu hình lograge
    
Tạo file cấu hình cho lograge riêng ở đây cho dễ quản lý:
`config/initializers/lograge.rb`

Nội dung cấu hình mình hay sử dụng thì như sau. (Mình sẽ giải thích cụ thể hơn ở bên dưới)

```ruby
Rails.application.configure do
  config.lograge.enabled = true

  config.lograge.ignore_actions = ['StatusController#index']
  config.lograge.formatter = Lograge::Formatters::Logstash.new
  config.lograge.keep_original_rails_log = true
  config.lograge.logger = ActiveSupport::Logger.new Rails.root.join('log/lograge.log')

  config.lograge.custom_payload do |controller|
    {
      host: controller.request.host,
      request_id: controller.request.request_id,
      remote_ip: controller.request.remote_ip,
      user_agent: controller.request.user_agent,
      referer: controller.request.referer
    }
  end

  config.lograge.custom_options = lambda do |event|
    exceptions = %w[controller action format authenticity_token]
    log_data = {
      host: event.payload[:host],
      request_id: event.payload[:request_id],
      remote_ip: event.payload[:remote_ip],
      user_agent: event.payload[:user_agent],
      referer: event.payload[:referer],
      exception: nil,
      params: event.payload[:params].except(*exceptions)
    }
    if (e = event.payload[:exception_object])
      exception_log = [e.message]
      exception_log += e.backtrace if e.backtrace.present?
      log_data[:exception] = exception_log.join("\n")
    end
    log_data
  end
end
```

- Sử dụng format JSON để dễ thu thập phân tích xử lý log, và mình sẽ vẫn giữ log mặc định của Rails song song với lograge.

```
  config.lograge.formatter = Lograge::Formatters::Logstash.new
  config.lograge.keep_original_rails_log = true
  config.lograge.logger = ActiveSupport::Logger.new Rails.root.join('log/lograge.log')
```

- Hệ thống thường có endpoint để LB check status, và vì không cần thiết log cho endpoint này nên ta sẽ ignore nó ra khỏi log như sau

```
  config.lograge.ignore_actions = ['StatusController#index']
```

- Những thông tin như host, ip của nguồn truy cập cũng rất giá trị nên ta sẽ lấy và log thêm chúng sử dụng custom_payload và custom_options! Ngoài ra message và trace của exception cũng quan trọng nữa.

```
  config.lograge.custom_payload do |controller|
    {
      host: controller.request.host,
      ...
    }
  end

  config.lograge.custom_options = lambda do |event|
    exceptions = %w[controller action format authenticity_token]
    log_data = {
      host: event.payload[:host],
      ...
    }
    if (e = event.payload[:exception_object])
      exception_log = [e.message]
      exception_log += e.backtrace if e.backtrace.present?
      log_data[:exception] = exception_log.join("\n")
    end
    log_data
  end
```

## Giới thiệu AWS tool: CloudWatch Logs Agent

Giả sử trong hệ thống bạn có N server, vì log được ghi ra file trên mỗi server nên suy ra bạn sẽ có N file logs.
Giờ bạn cần phân tích log đến hệ thống trong khoảng thời gian đêm qua đến sáng nay chẳng hạn, bạn sẽ đi tải về xử lý N files à? Nếu bạn định làm như vậy thì thật là quá thiếu tinh tế.

Giải quyết ra sao? Xin chia sẻ tiếp với mọi người ở part 2 với chủ đề Centralized logging.