---
title: "Encrypt credentials yaml file in your own way!"
date: 2020-10-02T15:30:52+09:00
draft: false
tags: [Rails, AWS, KMS, security, tech]
language: vietnamese
toc: true
authors: [chienkira]
cover: /blog/images/encrypt_credentials_yml.png
---

**Bình thường các bạn hay quản lý những thông tin nhạy (credentials) cảm như access key, api token... trong ứng dụng Rails ra sao? Rails có sẵn những chức năng gì giúp bạn làm việc đó? Và làm ra sao nếu một ngày, bạn gặp phải yêu cầu hệ thống quá khó và chức năng của framework không thể cứu vãn bạn??**

## 1. Các cách quản lý credentials có thể sử dụng trong Rails

### Rails version < 5.2

Nếu version Rails mà bạn đang sử dụng cũ hơn 5.2, trong ứng dụng mặc định ban đầu Rails sẽ chuẩn bị cho bạn 1 file là `config/secrets.yml`.

Nội dung của nó thì sẽ kiểu như sau: 

```yaml
development:
  secret_key_base: 3b7cd727ee24e8444053437c36cc66c3
  your_key: SOME_VALUE
```

Đây chính là file mà Rails tạo sẵn ra để làm chỗ cho bạn lưu những thông tin nhạy cảm vào. Rất đen là file này chỉ là file yaml bình thường, không được cung cấp chức năng mã hóa nên việc có nên sử dụng, có an toàn để commit nó lên git không là một dấu hỏi lớn!!

Tiện đây, có thể có người sẽ thắc mắc là "việc commit cả những file này lên git có tác dụng gì?", thì xin trả lời là nó có 2 merits chính:
- Commit trong git thì sẽ quản lý được version và rollback được.
- Vì ở trong git nên thay đổi trong credentials và source code ứng dụng sẽ đảm bảo được đồng bộ - tức được deploy đồng thời cùng nhau mỗi khi có release.

### Rails version >= 5.2

Rails luôn cải tiến vì thế từ version 5.2 trở đi file yaml sẽ được mã hóa và giải mã tự động cho bạn! Rails tuyệt vời!!

Tên của file yaml cũng được đổi từ `secrets.yml` thành `credentials.yml.enc` cho meaningful hơn. (*Đuôi .enc thể hiện là file này đã được encrypted!*)

Nguyên tắc hoạt động của nó như sau:
- Rails sẽ sử dụng file master.key để làm encrypt key, mã hóa nội dung credentials vào trong file credentials.yml.enc
- File master.key (ban đầu Rails sẽ sinh ra ngỗng nhiên) theo nguyên tắc là bị ignore khỏi git - hiển nhiên rồi
- Còn file credentials.yml.enc thì giờ đã an toàn để chúng ta commit lên git
- Ứng dụng rails khi khởi động, chỉ cần có file master.key Rails sẽ tự động tiến hành decrypt lại thông tin credentials cho chúng ta
- Khi muốn cập nhật thông tin credentials, Rails chuẩn bị cho chúng ta command `EDITOR="vi" bin/rails credentials:edit` để mở và edit credentials với editor

## 2. Nếu một ngày yêu cầu hệ thống đòi hỏi một thứ phức tạp hơn?

Chức năng mã hóa credentials trong Rails >=5.2 (và Rails 6.0.3 mới nhất hiện tại vẫn vậy) cơ bản là phục vụ tốt trong phần lớn trường hợp rồi.
Tuy nhiên trong dự án gần đây, mình gặp phải 1 security policy rất khoai sọ.
Nó yêu cầu tất cả các key mã hóa phải được rotate định kỳ! Có nghĩa rằng cất giấu kỹ càng file master.key và sử dụng là chưa đủ, hàng năm ta cần phải cập nhật lại nó, kéo theo là đồng thời cũng phải maintance lại file credentials.yml.enc. :cry:

Nếu sử dụng chức năng mặc định của Rails, sẽ phát sinh chi phí về người, thời gian để rotate key mã hóa và maintance các nội dung đã mã hóa hàng tháng, hàng năm...

Vì thế bài toán được đặt ra là mình phải tự tạo lấy cơ chế mã hóa/giải mã riêng đáp ứng yêu cầu dự án.

### Quản lý key mã hóa ở AWS KMS

*AWS KMS - Key manegement service*

Vì hệ thống sử dụng AWS rồi, service KMS của AWS lại có chức năng tự động rotate encrypt key nên xài cái này thôi!

### Dỡ bỏ cơ chế mã hóa credentials mặc định của Rails

1. Đầu tiên là xóa bỏ file config/credentials.yml.enc và config/master.key đi
2. Nói Rails không cần file master.key nữa vì chúng ta không sử dụng nó làm encrypt key nữa, bằng cách thêm dòng sau vào config/application.rb

    ```ruby
    config.require_master_key = false
    ```

### Thêm vào file mã hóa riêng của chúng ta

Để dễ sử dụng, file mã hóa mới chúng ta sẽ sử dụng file path hoàn toàn y hệt với spec mặc định của Rails. Có nghĩa là vẫn là file config/credentials.yml.enc nhưng mã hóa theo cách hoàn toàn khác.

Việc mã hóa/giải mã file yaml sử dụng KMS thì ta sẽ sử dụng gem có tên là [yaml_vault](https://github.com/joker1007/yaml_vault)

```
# In Gemfile file

# Encrypt yaml files
gem 'yaml_vault'
```

Để Rails đọc file config/credentials.yml.enc và giải mã credentials cho chúng ta sử dụng thì ta sẽ thêm đoạn xử lý sau vào application.rb

```ruby
config.yaml_vault_key_id = 'xxxx-xxxx-xxxx-xxxx'
decryped = YamlVault::Main.from_file(
  Rails.root.join('config/credentials.yml.enc').to_s, [['$']],
  'kms', aws_kms_key_id: config.yaml_vault_key_id
).decrypt_hash
config.credentials = decryped[Rails.env.to_s] || {}
```

Như vậy trong code của Rails app, bạn có thể sử dụng thông tin mã hóa trong credentials thông qua Rails.configuration.credentials như trước đây, hoàn toàn không thay đổi gì cả! :muscle:

### Thêm task để edit file credentials

Khi cần cập nhật credentials thì sao, chúng ta cũng muốn có một thứ giống cái command của Rails `rails credentials:edit` để chỉnh sửa trong editor đúng không.

Vì thế mình đã viết 1 command `rails credentials_yml:edit` tương tự như sau.

```ruby
# rubocop:disable Metrics/BlockLength
namespace :credentials_yml do
  desc '暗号化済のconfig/credentials.yml.encファイルを編集'

  task edit: :environment do
    decrypted_contents = YamlVault::Main.from_file(
      credentials_yml_file,
      [['$']],
      'kms'
    ).decrypt.to_yaml
    File.write(tmp_file, decrypted_contents)
    system("vi #{tmp_file}") # open tmp file with vi editor
    puts 'Waiting for credentials file to be saved. Abort with Ctrl-C.'

    encrypted_contents = YamlVault::Main.from_file(
      tmp_file,
      [['$']],
      'kms',
      aws_kms_key_id: Rails.configuration.yaml_vault_key_id
    ).encrypt.to_yaml
    File.write(credentials_yml_file, encrypted_contents)
    puts 'Saved credentials file'
  ensure
    FileUtils.rm(tmp_file) if File.exist?(tmp_file)
  end

  private

  def credentials_yml_file
    @credentials_yml_file ||= Rails.root.join('config/credentials.yml.enc').to_s
  end

  def tmp_file
    @tmp_file ||= File.join(Dir.tmpdir, 'credentials.yml').to_s
  end
end
# rubocop:enable Metrics/BlockLength
```

Bằng command trên, bạn có thể edit thông tin trong credentials bằng VI editor và sau khi save nội dung, file credentials sẽ tự động được mã hóa lại cho bạn sẵn sàng commit lên git :))

**Hi vọng chia sẻ sẽ giúp ích ít nhiều hoặc cho bạn ý tưởng để giải quyết những vấn đề đau đầu mà anh em dev chúng ta ngày nào cũng gặp nhé!**
