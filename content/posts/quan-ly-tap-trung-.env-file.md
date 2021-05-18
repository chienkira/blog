---
title: "Centralize .env file manegement"
date: 2021-05-18T13:03:41+09:00
draft: false
tags: [aws, codebuild, parameter store, devops]
language: vietnamese
toc: true
authors: [chienkira]
cover: /blog/images/__Centralize__ .env file management.png
---

**Như các bạn đã biết rồi, đa số các ứng dụng backend lẫn frontend hiện nay đều sử dụng .env file để lưu trữ các biến số phụ thuộc vào môi trường (môi trường prd or stg or dev or local). Khi bắt tay vào cài đặt CI/CD chúng ta mới nhận ra là, làm sao để load được .env file một cách tự động ngay trong pipeline CI/CD?? Bài viết này xin chia sẻ cách làm của mình, các bạn tham khảo thử nhé!**

## Lưu trữ ở đâu?

Mình sử dụng parameter store - dịch vụ tuyệt vời của AWS hỗ trợ các case cần lưu trữ và quản lý tập trung.

Đặc điểm nổi bật của parameter store:

- có thể chọn 2 kiểu lưu trữ, String cho dữ liệu thông thường hoặc SecureString nếu cần mã hoá và bảo mật hơn. Ngoài ra có kiểu StringList nhưng cá nhân mình thấy ít khi cần dùng đến.
- key của param là dạng path (ví dụ `/docker-hub/userid`) nên có thể tổ chức hoá - hierarchy, cho phép dễ dàng grouping để quản lý cũng như truy xuất


### Cách làm đơn giản

Quay lại vấn đề làm sao để quản lý file .env trên parameter store, cách đơn giản nhất thì như bạn đang mường tượng ra rồi đó - cứ thế mà lưu trữ nội dung cả file .env vào parameter store thôi.
Cách làm này đơn giản nên có ưu điểm là lấy ra và phục hồi lại file .env rất dễ dàng.

Giả sử nếu chúng ta lưu file .env cho môi trường staging dưới key path là `/service-name/env/stg`, chỉ cần 1 câu lệnh sau là có thể khôi phục ra file .env rồi.

```
$ aws ssm get-parameters-by-path --path /service-name/env/stg --with-decryption > .env
```

Tuy nhiên nếu như file .env của dự án của bạn quá lớn thì khả năng sẽ không thể xài cách này được nữa. Lý do là parameter store có áp đặt upper limit 4096 characters cho 1 param, không thể lưu 1 đoạn text dài hơn giới hạn đó được.

### Cách làm better

Để đảm bảo theo thời gian khi file .env phình to hơn, bạn vẫn làm việc ngon ơ mà không phải thay đổi gì thì bạn cần một cách lưu trữ "thông minh" hơn.

Sau đây là cách làm của mình:

- mỗi giá trị cấu hình trong file .env sẽ được lưu thành 1 param độc lập
- key path của param sẽ được tổ chức theo từng môi trường. Ví dụ với một giá trị cấu hình API_URI, ứng với môi trường production và staging mình sẽ lưu 2 param là `/service-name/env/stg/API_URI` và `/service-name/env/prd/API_URI`
- khi cần khôi phục file .env cho môi trường nào thì chỉ cần load toàn bộ các param ở dưới key path của môi trường đó. Ví dụ môi trường staging thì sẽ là `/service-name/env/stg/*`

Trong lần đầu để upload toàn bộ cấu hình lên parameter store, số lượng khá nhiều nên nếu như dùng AWS web console thì sẽ khá mỏi tay. Recommend bạn dùng aws cli cho việc này nhé.
Ex: 
```
aws ssm put-parameter --overwrite --region ap-northeast-1 --name "/seminar-web/env/stg/API_URI" --type "String" --value "https://foobar.io"
```

![](/blog/images/Screen Shot 2021-05-18 at 14.09.55.png)

## Implement thực tế vào CodeBuild

Mình chia sẻ tiếp cách phục hồi file .env từ trong CodeBuild.
※ Trước tiên bạn đảm bảo IAM role mà CodeBuild được attach có quyền ssm:GetParameters và ssm:GetParametersByPath nhé. Cái này mình cũng rất hay quên..

Trong pre_build phase, mình sẽ xác định xem cần build cho môi trường production hay staging, dựa trên git branch được push lên.
Ứng với mỗi môi trường, mình khai báo sẵn prefix key path mà cover được tất cả các giá trị cấu hình của môi trường đó.

```
phases:
  pre_build:
    commands:
      - GIT_BRANCH=$(echo $CODEBUILD_WEBHOOK_HEAD_REF | cut -c 12-)
      - |
        if [ $GIT_BRANCH == "master" ]; then
          echo Target environment is PRODUCTION
          PARAMTER_PREFIX=/seminar-web/env/prd/
        else
          echo Target environment is STAGING
          PARAMTER_PREFIX=/seminar-web/env/stg/
        fi
    finally:
      - echo === PRE_BUILD finished on `date`
```

Sau đó ở phase build, sử dụng get-parameters-by-path với tham số path là prefix ở bên trên, mình load ra toàn bộ các giá trị cấu hình cần thiết.
Lưu ý sử dụng --query và --output để chuẩn hoá nội dung đầu ra thành dạng `/seminar-web/env/stg/API_URI = https://foobar.io`.

```
phases:
  build:
    commands:
      - echo === BUILD started on `date`
      # Generate .env file
      - aws ssm get-parameters-by-path --path $PARAMTER_PREFIX --query "Parameters[*].[Name,'=',Value]"
        --region $AWS_DEFAULT_REGION --output text | sed 's/[[:blank:]]//g' | sed "s|$PARAMTER_PREFIX||g" > ./.env.local
```

Vì mục đích cuối cùng file .env phải là dạng `API_URI=https://foobar.io` cho nên mình cần pipeline tiếp cho sed để xử lý.

- `sed 's/[[:blank:]]//g'` để xoá cách dấu spaces xen giữa.
- `sed "s|$PARAMTER_PREFIX||g"` để xoá bỏ phần prefix (/seminar-web/env/stg/) thừa ở đầu mỗi dòng.
Chú ý ở đây mình dùng separator `|` chứ không phải `/` vì bản thân $PARAMTER_PREFIX có gồm các ký tự `/` rồi nên phải làm vậy để tránh lỗi.