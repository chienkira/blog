---
title: "Explain Mixin and How to use it with Python"
date: 2019-06-25T10:25:00+09:00
draft: no
tags: [python, programing, mixin]
language: vietnamese
toc: false
authors: [chienkira]
---

**Vì thường ưu tiên tốc độ, chỉ tập trung làm sao đạt được output mong muốn trong thời gian ngắn nhất, mà từ bao giờ mình đã quá quen với việc "tái sử dụng" vô tội vạ các mã code Google được. Dẫn đến ngày nọ, mình nhận ra mình hơi bị mất niềm tin vào chính bản thân. Để khắc phục điều này, mình quyết định lâu lâu sẽ ôn tập củng cố lại kiến thức lập trình cơ bản, nhất là những tính năng độc đáo có chất đặc thù một tí của từng ngôn ngữ. Lần này chủ đề sẽ là Mixin và Python!**

## Mixin là gì

#### Đầu tiên rất là cơ bản vỡ lòng thôi, cách đọc của "mixin"!

Mình khá là gà khi một thời gian đầu cứ nghĩ rằng nó đọc là "mờ i mi, sờ in sin, mi sin!".
Đây là cách đọc sai nhé ạ, mặc dù dễ đọc với mình ra phết!
Để đọc đúng, đầu tiên chúng ta phải hiểu rằng bản chất mixin nó được ghép từ 2 từ tiếng anh là "mix" và "in".
Vì thế cách đọc chuẩn phải là "mích in" :joy: nghĩa nôm na là "nhào trộn vào" nha!

#### Giải thích Mixin 

Mixin xuất hiện trong nhiều hình dạng khác nhau với từng ngôn ngữ lập trình, nhưng điểm chung của nó là đều ám chỉ về một kỹ thuật giúp tái sử dụng code, khiến code ngắn gọn cô đọng hơn.

Một ví dụ sử dụng mixin ở trong SASS(CSS) như sau. Code ngắn hơn, khoa học hơn và đảm bảo khi code bự ra thì cách dùng mixin sẽ dễ sửa chữa hơn nhá!

```CSS
/* Định nghĩa mixin size */
@mixin size($width, $height: $width) {
  width: $width;
  height: $height;
}

/* Ví dụ sử dụng mixin size */
.tiny-box {
  @include size(10px, 20px);
}
.huge-box {
  @include size(1000px, 2000px);
}
```

Tuy nhiên cách hiểu rộng rãi và phổ biến nhất của mixin thì lại được biết đến thông qua các ngôn ngữ lập trình hướng đối tượng. Nhắc đến mixin, người ta thường nghĩ đến kỹ thuật để thêm vào nhiều class một tập các thuộc tính/method sẵn có.

Và đặc biệt Python với việc hỗ trợ tính năng đa kế thừa, thực sự là một đất diễn cho các cao thủ nào mê mẩn "mích in" :smiley: 

- *Đa kế thừa có nghĩa là một class có thể khai báo kế thừa cùng 1 lúc nhiều hơn 2 class cha, so sánh với các ngôn ngữ khác như Java hay PHP và cả Ruby chỉ có đơn kế thừa mà thôi.*

- *À riêng Ruby là một ngôn ngữ mình khá yêu thích do đó mình đã tìm hiểu qua để minh oan cho em nó. Ruby có cách làm riêng của ruby để cho các anh chị tha hồ mà "mích in" nhé, cụ thể là khai báo module rồi include nó vào class ạ. Ruby không cùi đâu! :))*

Ví dụ có code cho dễ hình dung thôi nào. Gì thì theo mình, anh em lập trình 
nói chuyện với nhau qua dòng code vẫn là cách truyền đạt hiệu quả nhất!

---

Đầu tiên ta khai báo một mixin, thực chất là một class. Chúng ta nên đặt tên class kết thúc với từ khóa Mixin, để tránh sau này khi đa kế thừa không biết đâu là đang kế thừa class đâu là đang ốp mixin vào nữa!

```python
import logging


class LoggerMixin(object):
    @property
    def logger(self):
        name = '.'.join([
            self.__module__,
            self.__class__.__name__
        ])
        return logging.getLogger(name)

    def log_exception(self, e):
        self.logger.error(str(e))
```

Mục đích của mixin này là cung cấp một thuộc tính `logger` và một method `log_exception`.

Giờ ta thử sử dụng mixin trên.

```python
class FirstClass(LoggerMixin, BaseClass):
    def do_some_thing(self):
        try:
            raise Exception("oh no")
        except Exception as e:
            self.log_exception(e)


class AnotherClass(LoggerMixin, BaseClass):
    def do_another_thing(self):
        try:
            raise Exception("oh no!!!")
        except Exception as e:
            self.logger.debug("don't worry!")
```

2 class `FirstClass` và `AnotherClass` sử dụng `LoggerMixin` nên được thừa hưởng "khối tài sản khổng lồ" và suốt đời tha hồ `self.logger` hay `self.log_exception` :)) Tuyệt vời không các bạn, giờ các bạn có thể tùy thích "mích" LoggerMixin vào mọi class mà bạn muốn. Và vì là đa kế thừa, nên ta có thể "mích" nhiều hơn 2 mixin cùng một lúc, không chỉ vậy vừa "mích in" ta vẫn vừa có thể kế thừa các class cha nữa! Python xịn xò quá hà! :joy:

## Chú ý khi sử dụng Mixin với Python

Giờ giả sử có một bạn mà style hơi oái oăm kiểu "thích thì làm thôi", bạn ấy sử dụng nhiều mixin mà trong khi các mixin lại có trùng tên method với nhau, thế thì thử hỏi Python sẽ sử dụng method của mixin nào???

Để giải đáp cái này mình đã tìm hiểu và phát hiện ra trong đa kế thừa, Python xử lý theo thứ tự hơi ngược so với những gì bình thường ta tưởng.
Điều này không chú ý có thể khiến code của chúng ta tạo ra bug.

Ta có đoạn code ví dụ sau:
```python
class Mixin1(object):
    def test(self):
        print "Mixin1"

class Mixin2(object):
    def test(self):
        print "Mixin2"

class MyClass(BaseClass, Mixin1, Mixin2):
    pass

obj = MyClass()
obj.test()
```

Theo thứ tự kế thừa `(BaseClass, Mixin1, Mixin2):` thì chắc hẳn Mixin2 sẽ overide Mixin1 và kết cục là method test() của MyClass sẽ là method của Mixin2?!? Nếu các bạn nghĩ vậy thì các bạn đã giống mình - sai hoàn toàn.

Kết quả của đoạn code trên là:
```python
>>> obj = MyClass()
>>> obj.test()
Mixin1
```

Python hiểu khai báo đa kế thừa từ **phải qua trái**, ngược lại so với thứ tự tự nhiên khi con người đọc chữ. Do đó cách khai báo đúng phải là như sau:
```python
class MyClass(Mixin2, Mixin1, BaseClass):
    pass
```

Thứ tự này lúc đầu hơi khó quen nhưng một khi nhận ra rằng nó chính là thứ tự của các class đang xếp chồng lên nhau thì mọi việc dễ dàng hơn khá nhiều:
`MyClass => Mixin2 => Mixin1 => BaseClass`

Các bạn chú ý nha!

