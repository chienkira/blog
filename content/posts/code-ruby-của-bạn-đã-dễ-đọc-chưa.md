---
title: "Code Ruby Của Bạn Đã Dễ Đọc Chưa"
date: 2019-10-24T09:52:45+09:00
draft: false
tags: [ruby, rails, code-style, programing, tech, til]
language: vi
toc: true
authors: [chienkira]
cover: /blog/images/code_readable.png
---

**Do you care about writing readable code? Bài này mình muốn giới thiệu tới các bạn vài mẹo mà thời gian gần đây mình mới học được, giúp cho code Ruby trông chuyên nghiệp, dễ đọc hơn. Đây đều là những tip mà cá nhân mình thấy rất hay nhưng lại không nhiều người biết đến hoặc áp dụng thực tế. Nếu bạn cũng có tips muốn chia sẻ, comment bên dưới nhé ↓↓↓ :smiley:**

## 1. Dùng underscore để viết số "bự" dễ đọc hơn

Đố nhanh! có bao nhiêu số 0 trong số sau `1000000000000` ?

Hết giờ rồi, bạn đã đếm xong chưa??

Mình đoán có hai khả năng, 1 là bạn thử cố đếm và chưa đếm xong, 2 là bạn chả buồn đếm mà đang làu bàu: "dỗi hơi vãi, sao mị không viết kiểu `1,000,000,000,000` cho dễ đọc mà bày trò đố với chả đoán?!"

Yes! Ước gì Mỹ Tâm có thể viết như thế trong code cho dễ hiểu ha. Nhưng mà đáng tiếc, dấu phẩy lại là ký tự quan trọng được quy định trong rất nhiều biểu thức/khai báo của các ngôn ngữ lập trình rồi, do đó bạn mà viết `1,000,000,000,000` thì hỏng bét ngay! Tuy nhiên các ngôn ngữ thánh thần đã bày cho chúng ta một cách viết tương tự - sử dụng underscore.

```ruby
# bad - how many 0s are there?
num = 1000000

# good - much easier to parse for the human brain
num = 1_000_000
```

Cái mẹo này là cái mình thấy hay mà lại trước giờ chả biết đến, đâm ra rất là khoái nó, hôm nay xài để refactor code dự án đang làm luôn :))

Ngoài ra mình check thử thì python 3.6 trở đi cũng hỗ trợ cách viết này, JS (không rõ từ ES mấy) cũng thấy đang hỗ trợ rồi. Nhớ áp dụng nhé mọi người!

## 2. Tên của method trả về boolean thì nên thêm question mark (?)

Có nhiều cách để "thông báo" cho người khác biết rằng, method của tôi này đang kiểm tra điều kiện gì đó rồi trả về giá trị true/false. Một trong số đó là sử dụng tiền tố `is` vào trong method name - cách phổ biến được sử dụng cho nhiều ngôn ngữ lập trình.

Tuy nhiên ruby có cách viết sáng tạo hơn là đặt `?` vào cuối method name. Cá nhân mình thấy cách này hơn, vì ngắn gọn - không sử dụng thêm tiền tố, và giống đang viết tiếng anh hơn là viết code nên cực kỳ dễ đọc. LoL

```ruby
# bad
class Person
  def is_tall
    height > 170
  end
end

# good
class Person
  def tall?
    height > 170
  end
end
```

Ngoài ra còn có style tương tự là thêm `!` vào cuối tên method mà bạn muốn ngụ ý rằng "cần cẩn trọng" khi sử dụng.

Những style này đơn giản, thấy được giới thiệu ở blog tứ phương, và sẽ bắt gặp suốt ngày nếu bạn code Rails nhưng mà mình lại chưa thấy nhiều người xung quanh chủ động áp dụng được nó khi tự phải viết class hay module gì đó. Mình cũng phải nhớ để thói quen hóa cái này mới được!

## 3. Dùng tiền tố _ cho biến không dùng đến

Chắc bạn đã từng bắt gặp những trường hợp như:

- xử lý hash, nhưng chỉ cần làm việc với value, bỏ qua key
- method trả về nhiều giá trị nhưng bạn chỉ cần 1 trong số chúng

Bạn xử lý ra sao? Mình trước đây đều pick đại một cái tên và tạo ra biến mới để lưu những
giá trị không sử dụng đó.  Kiểu như dưới đây này ↓

```ruby
# bad
unused_var, want_to_use_var = something_else(x)
do_somthing_else(want_to_use_var)
```

Cách làm này dẫn đến 2 vấn đề, một là code vẫn dài mà nhìn vào lại
khó hiểu đâu mới là giá trị đoạn code đang cố xử lý, hai là biến tạo ra không dùng sẽ để lại
cảnh báo kiểu "Variable set but not used".

Vậy Ruby có gì cho chúng ta trong trường hợp này? Vâng lại là underscore `_` thần thánh :))
Chúng ta được khuyến khích sử dụng `_` cho mọi thứ mà không muốn dùng đến - nhưng vẫn phải viết ra để 
đảm bảo tuân thử syntax của ngôn ngữ. `_` có thể dùng trong cả block lẫn khi gán trị cho biến như 2 ví dụ dưới đây.

```ruby
# good
# When want to ignore keys, use only values of a hash
result = hash.map { |_, v| v + 1 }

# When only want to use the second variable
_, want_to_use_var = something_else(x)
```

## Kết
Còn vô vàn code style khác bạn có thể tham khảo ở đây.

- bản gốc tiếng anh [rubocop/ruby-style-guide](https://github.com/rubocop-hq/ruby-style-guide)
- bản dịch tiếng Việt [rubocop/ruby-style-guide vietnamese](https://github.com/CQBinh/ruby-style-guide/blob/master/README-viVN.md)

Ngoài ra, mình khuyến khích cài plugin để format code (nếu dùng editor) hoặc sử dụng IDE có chức năng format code. Bằng cách đó, mấy vụ cơ bản để làm code "sạch sẽ" ví dụ như đặt space ra sao, brackets dùng thế nào, khi nào dùng one line... sẽ auto được làm cho rồi. Đặc biệt nếu dùng RubyMine thì code inspection của nó còn hoàn toàn tương thích với bộ style của rubocop trong link phía trên, nên code bậy bạ là nó "chửi" cho ngay, một thời gian sau sẽ thuộc và tiến bộ thấy rõ :joy: