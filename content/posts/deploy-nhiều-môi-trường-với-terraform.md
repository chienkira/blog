---
title: "Áp Dụng Terraform Module deploy lên nhiều môi trường"
date: 2019-06-15T10:28:15+09:00
draft: false
tags: [aws, terraform, SRE, infra]
language: vi
toc: true
authors: [chienkira]
cover: /blog/images/terraform-module.png
---

**Vào mùa mưa rồi, ngày cuối tuần bị nó hành cho suốt luôn. Vừa mưa còn vừa lạnh, ai nghĩ đây là mùa hè chứ!!. Tuy nhiên cũng nhờ thế mà nhớ đến cái sở thích 地味 này :joy: Trong bài viết này, mình muốn giới thiệu về *module* trong *terraform*, và cách sử dụng nó để deploy lên nhiều môi trường dev-stg-prod khác nhau.**

## TL;DR

- Các lệnh của terraform mặc định chỉ xử lý các file *.tf ở cùng thư mục hiện tại
- Bằng cách sử dụng *module*, chúng ta có thể tham chiếu và tái sử dụng các file *.tf ở thư mục/server khác
- Kết hợp sử dụng *variables*, phía gọi *module* có thể override các thiết lập được cài đặt trước đó trong *module*
- Mỗi môi trường dev-stg-prod thì kiểu gì cũng có các thiết lập khác nhau (ví dụ domain khác nhau, spec của EC2 hay container khác nhau...) nhưng lại phải dựa trên một thiết kế infra đồng nhất. Do đó *module* hóa infra code là một cách hiệu quả để giữ cho code sạch sẽ, DRY và dễ quản lý.

## Giải thích cơ bản terraform module
- Module như đúng tên của nó, ám chỉ việc chúng ta gom configurations lại và tái sử dụng ở nhiều nơi/nhiều lần.
- Module thực sự là một trong những tính năng ăn điểm của Terraform. Khi mà bạn muốn tạo ra nhiều môi trường có cấu trúc infra tương tự nhau, ban đầu bạn có thể nghĩ đến việc copy paste source rồi `terraform apply`, rất nhanh và đơn giản! Nhưng về lâu dài thử tưởng tượng đến việc thay đổi một vài chỗ sẽ khiến bạn phải sửa chữa cả tá vị trí bị copy paste kia, chưa kể đến việc làm vậy sẽ tiềm ẩn rủi do operation miss... chắc chắn bạn sẽ muốn module hóa và có một DRY infra code sạch sẽ!
- Cách tạo module thì rất đơn giản.
    - Đầu tiên chúng ta cần tạo một thư mục, trong đó định nghĩa các resource cần thiết của module. Ví dụ mình tạo thư mục `awsome_module`, rồi tạm thời mình tạo một resource như sau:
    {{< highlight yaml >}}
    # ./awsome_module/sqs.tf
    resource "aws_sqs_queue" "search_queue" {
        name = "search-queue.fifo"
        fifo_queue = true
        content_based_deduplication = true
    }
    {{< / highlight >}}

    - Sau đó, ở chỗ cần sử dụng module chúng ta chỉ cần định nghĩa nó với từ khóa `module` và `source`, `source` để chỉ định relative path tới thu mục của module. Ví dụ như sau:
    {{< highlight yaml >}}
    # ./main.tf
    module "my_module" {
        source = "./awsome_module"
    }
    {{< / highlight >}}

    - Chỉ vậy thôi! Thử init ở thư mục chứa `main.tf` rồi chạy plan,chúng ta sẽ thấy terraform nhận biết cả resource SQS được định nghĩa trong `./awsome_module/*`.
    {{< highlight bash >}}
    $ terraform plan
    ...
    Terraform will perform the following actions:

    # module.my_module.aws_sqs_queue.search_queue will be created
    + resource "aws_sqs_queue" "search_queue" {
        + arn                               = (known after apply)
        + content_based_deduplication       = true
        + delay_seconds                     = 0
        + fifo_queue                        = true
        + id                                = (known after apply)
        + kms_data_key_reuse_period_seconds = (known after apply)
        + max_message_size                  = 262144
        + message_retention_seconds         = 345600
        + name                              = "search-queue.fifo"
        + policy                            = (known after apply)
        + receive_wait_time_seconds         = 0
        + visibility_timeout_seconds        = 30
        }

    Plan: 1 to add, 0 to change, 0 to destroy.
    {{< / highlight >}}

## Implement thực tế

### Yêu cầu đặt ra là gì?

- Mình sẽ muốn tạo ra 3 môi trường dev-stg-prod với cấu trúc infra giống hệt nhau. Tuy nhiên thông số thiết lập thì sẽ khác nhau, ví dụ rất thực tiễn nhé, dev và stg env mình muốn để cấu hình Fargate thấp thôi cho tiết kiệm tiền :smiley: còn riêng prod thì ưu ái cho cấu hình xịn xò trên một bậc chẳng hạn.
- State của terraform thì phải được lưu trữ "trên mây". Team nhiều người dev/deploy để làm product thật chứ có phải là chơi bời linh tinh đâu! → Cần phải định nghĩa backend của terraform lên S3 cho đàng hoàng.
- Code infra phải DRY, ví dụ khi cần thay đổi/thêm bớt resource nào đó thì chỉ cần sửa code một nơi thôi là có thể apply lên cả 3 môi trường.

### Áp dụng module hóa, xử lý các yêu cầu trên

Đi vào thẳng vấn đề, dưới đây là cấu trúc code mình đã dựng lên.

```bash
    |--terraform 　　　　　　　　　# infra code root directory
    |  |--_module　　　　　　　　　# shared resource definitions
    |  |  |--ecr.tf
    |  |  |--ecs.tf
    |  |  |--iam.tf
    |  |  |--log.tf
    |  |  |--network.tf
    |  |  |--...etc...
    |  |  |--s3.tf
    |  |  |--sqs.tf
    |  |  |--variables.tf
    |  |--dev                   # configurations for DEV env
    |  |  |--backend.tf
    |  |  |--main.tf
    |  |--stg                   # configurations for STG env
    |  |  |--backend.tf
    |  |  |--main.tf
    |  |--prod                  # configurations for PROD env
    |  |  |--backend.tf
    |  |  |--main.tf
```

Để đảm bảo infra code DRY, mình chọn cách gom tất cả các resource cần thiết vào làm thành một module. Chính là thư mục `_module` ở trong tree bên trên.

- 1 nguyên tắc quan trọng ở đây là, toàn bộ thiết lập trong các resource nên tham chiếu đến `var`, không fix cứng giá trị! Việc này nhằm để sau này chúng ta có thể tối đa hóa khả năng custom thiết lập theo từng môi trường.
- Ví dụ như trong file `_module/ecs.tf` mình sẽ định nghĩa ECS task như dưới đây, thông số vCPU và memory của container tham chiếu đến `var.fargate_cpu` và `var.fargate_memory`.
    {{< highlight yaml >}}
    ...
    resource "aws_ecs_task_definition" "task" {
        family = "${var.env}-${var.product}-${var.app}-search-task"
        execution_role_arn = "${aws_iam_role.task_role.arn}"
        network_mode = "awsvpc"
        requires_compatibilities = ["FARGATE"]
        cpu = "${var.fargate_cpu}"
        memory = "${var.fargate_memory}"
    ...
    {{< / highlight >}}

- Các `var` cần sử dụng trong module mình sẽ tổng hợp lại trong file `variables.tf`. Ở đây mình sẽ chỉ khai báo sự tồn tại của biến thôi, không định nghĩa giá trị gì hết vì sau đó lúc gọi và sử dụng module, mình mới muốn thiết lập thông số cụ thể.
    {{< highlight yaml >}}
    variable "fargate_cpu" {}
    variable "fargate_memory" {}
    {{< / highlight >}}

Tiếp theo mình có 3 thư mục ứng với 3 môi trường dev-stg-prod. Trong mỗi thư mục có 2 file.

- backend.tf: Như đã trình bày ở trên, một yêu cầu bắt buộc là state phải được lưu trên remote. Do đó cần định nghĩa `backend` cho terraform, cụ thể mình sẽ lưu trên AWS S3. Chú ý là `backend` nằm trong xử lý khởi tạo của terraform, do đó terraform không cho phép sử dụng tham chiếu `var` hay `local` ở đây. Mình lại muốn sử dụng 3 bucket khác nhau cho 3 môi trường nên mới xuất hiện 3 file backend.tf như thế này.

- main.tf: Khai báo sử dụng module ở thư mục `_module` đồng thời báo với terraform các giá trị thiết lập mong muốn khi sử dụng module đó. Ví dụ ở môi trường prod mình muốn ECS task có 1vCPU và 2GB memory, nhưng ở môi trường dev thì chỉ cần 0.5vCPU và 1GB memory thôi.

    `prod/main.tf`
    {{< highlight yaml >}}
    module "app" {              # use module
        source = "../_module"   # at '../_module' folder

        fargate_cpu = "1024"    # 1 vCPU = 1024 CPU units
        fargate_memory = "2048"
    }
    {{< / highlight >}}

    `dev/main.tf`
    {{< highlight yaml >}}
    module "app" {              # use module
        source = "../_module"   # at '../_module' folder

        fargate_cpu = "512"
        fargate_memory = "1024"
    }
    {{< / highlight >}}

Với cách dựng code trên, thao tác deploy lên các môi trường sẽ như sau.
Ví dụ ta đang muốn deploy lên môi trường dev.
```bash
$ cd dev         # cd to folder of deploy-target env
$ terraform init
$ terraform apply
```

## Kết luận
Sử dụng `module` giúp infra code viết bằng terraform trở lên DRY sạch sẽ, mà vẫn đảm bảo tính độc lập cho phép chúng ta thiết lập các thông số khác nhau giữa các môi trường.

Thực sự về lý thuyết thì cách sử dụng này không phải là cách sử dụng vốn được thiết kế dành cho `module`. Mình đã đọc doc của terraform và cũng biết rằng, tính năng `workspace` mới là tính năng được gọi ý sử dụng nếu muốn deploy lên nhiều môi trường. Tuy nhiên sau khi thử sử dụng thì mình hoàn toàn không có cảm tình với `workspace`. Lý do là khi sử dụng `workspace`, việc đọc giá trị thiết lập ứng với từng workspace dev-stg-prod làm cho code terraform trở lên dài và khó đọc hơn khá nhiều, do phải sử dụng `if` hoặc hàm `lookup`. Đối với mình và team hiện tại, mọi người đều chung ý kiến rằng cách dựng code sử dụng module trên là dễ hiểu và dễ sử dụng hơn cả, do đó thực tế thì mình cũng không sử dụng `workspace`. Tương lai nếu sử dụng đến tính năng gì đó mà chỉ có workspace mới hỗ trợ thì chắc sẽ chuyển qua sau vậy?! :smiley:



