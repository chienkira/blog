---
title: "Better Logging Practice (Part 2)"
date: 2020-07-30T10:00:38+09:00
draft: true
tags: [rails, logging, tech, lograge, aws]
language: vi
toc: true
authors: [chienkira]
cover: /blog/images/Centralized_logging_aws_cloudwatch_agent.png
---

**Đại dịch covid-19 tiến hóa thành covid-20 và cả ở Nhật hay ở Việt Nam thì tình hình đều đang rất đáng lo ngại. Bên Nhật còn vài tháng nữa là lại vào mùa lạnh, mùa cúm infu nữa... không biết sẽ ra sao. Thôi không đánh trống lảng nữa, lý do mãi mới viết nốt part 2 là do lười ham chơi thôi! :smile: :pray: Let's get start!!**

## Giới thiệu AWS Cloudwatch agent

Như đã giới thiệu ở bài viết phần 1 trước đó -  [Better logging practice (Part 1)](https://chienkira.github.io/blog/posts/better-logging-practice/), giả sử service của chúng ta chạy trên nhiều servers, vậy thì sẽ dẫn đến vấn đề là logs bị phân tán ra nhiều nơi đúng không?!

Đối mặt với nó, dĩ nhiên ta có thể chấp nhận và chọn chả làm gì cả. Ngày xưa lúc làm việc với service chạy trên 2 servers thì thực ra mình cũng "chấp nhận" như thế. Khi nào cần xem thì scp tải logs từ 2 servers về, so khớp thời gian rồi ráp vào làm 1 xong "ngắm" :joy: Các bạn nào biết rồi đừng chửi mình "nông dân" nhé (^^)!

Tuy nhiên, ở bài viết này mình muốn giới thiệu với các bạn cách giải quyết chuyên nghiệp hơn - đó là kỹ thuật centralized logging. Chân lý của centralized logging cũng được giới thiệu trong 12 factor (https://12factor.net/logs) đấy nhé, huyên nghiệp lắm!

> In staging or production deploys, each process’ stream will be captured by the execution environment, collated together with all other streams from the app, and routed to one or more final destinations for viewing and long-term archival

Chân lý nói chúng ta rằng logs nên được stream tập trung đến 1 vài (tốt nhất là 1) nơi lưu trữ cuối cùng, phục vụ việc xem và cất trữ logs. Quá hợp lý phải không ạ. Và Cloudwatch agent chính là tool mà AWS cung cấp cho chúng ta để hiện thực hóa chân lý trên. 
Tính năng của Cloudwatch agent gồm có:
- Chạy dưới dạng service trên linux OS trong server của chúng ta
- Tự động stream các file log đến AWS cloudwatch log group (ta có thể cấu hình chi tiết stream file nào đến log group/log stream nào)

## Cách cài đặt AWS Cloudwatch agent

Có vài cách để cài đặt cloudwatch agent, các bạn tham khảo hướng dẫn chính thức của AWS ở [đây](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html) để tìm ra cách thích hợp nhất với hệ thống của bạn.

Còn ở dưới đây mình tóm lược lại các bước cài đặt bằng command line cho server chạy Cent OS. Lưu ý là nếu đang sử dụng Ansible để tự động provision server thì các bước cài đặt cũng y hệt giống như dưới đây nhé.

```bash
# Tải và cài đặt package cloudwatch agent
$ cd ~ && wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
$ sudo rpm -U ./amazon-cloudwatch-agent.rpm

# Tạo file cấu hình (nội dung file cấu hình mình giải thích ở dưới)
$ sudo vi /opt/aws/amazon-cloudwatch-agent/bin/config.json

# Khởi động service cloudwatch agent
$ sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s
# Kiểm tra xem service chạy lên thế nào rồi cho chắc :))
$ sudo systemctl status -l amazon-cloudwatch-agent.service
```

File cấu hình cho cloudwatch agent là cách duy nhất để bạn thông báo cho agent biết phải stream log nào đến đâu.
Định dạng là json và dưới đây là 1 ví dụ đơn giản các bạn tham khảo nhé, nó cấu hình để stream nginx log thôi.

```json
{
  "agent": {
    "run_as_user": "deploy"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/nginx/foobar.nginx.access.log",
            "log_group_name": "foobar.nginx.access.log",
            "log_stream_name": "{instance_id}"
          },
          {
            "file_path": "/var/log/nginx/foobar.nginx.error.log",
            "log_group_name": "foobar.nginx.error.log",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}

```

Ngoài ra để agent chạy trên server có quyền stream logs vào cloudwatch thì chúng ta cần cung cấp cho nó role cần thiết. Cách tạo role tham khảo ở [đây](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/create-iam-roles-for-cloudwatch-agent-commandline.html) nhé!

## Check thành quả

Sau khi cài đặt và khởi động agent rồi, mở cloudwatch ra các bạn sẽ thấy logs được stream vào log group, độ trễ của log trên cloudwatch và log thật mình thấy thực tế chỉ khoảng 2 ~ 3s mà thôi. Rất cool phải không :))

![](/blog/images/cloudwatch_agent.png)

Dù hệ thống có kiến trúc đơn giản, log không bị phân tán thì mình vẫn khuyến khích cài đặt aws cloudwatch agent. Thay vì phải dùng những công cụ tốn kém để tổng hợp phân tích log thì dùng cloudwatch agent miễn phí cũng có thể stream log vào cloudwatch cho chúng ta. Ta có thể xài Logs Insights để query log trong những tính huống đơn giản, hay phức tạp thì xài Athena vào tha hồ xử lý. Xem log cũng không phải vào server, xem trên cloudwatch trực quan nhanh hơn nhiều. 

Nói chung merit vô vàn không kể hết :smiley: Chúc các bạn cài đặt được logging chuyên nghiệp dễ sử dụng hơn nhé!