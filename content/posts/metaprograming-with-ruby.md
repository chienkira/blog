---
title: "Metaprogramming In Ruby"
date: 2020-12-29T22:24:49+09:00
draft: false
tags: [ruby, programing, tech, rails]
language: vietnamese
toc: true
authors: [chienkira]
cover: /blog/images/__Metaprograming__ in __Ruby__.png
---

**Bài viết cuối năm 2020, mình muốn giới thiệu đến với các bạn kỹ thuật metaprogramming thông qua 1 ví dụ thực tiễn - viết Rest API nhé. Viết code ruby đã fun, có metaprogramming lại còn fun hơn!**

## What is metaprogramming?

Một cách ngắn gọn, metaprogramming là kỹ thuật để làm sourcecode - bản thân nó tự định nghĩa ra thêm code mới trong lúc runtime.
Tưởng tượng như bạn đang code để lập trình ra thêm những đoạn code khác vậy.
Nói như vậy nhưng không phải áp dụng nó để chúng ta tạo ra thứ gì bất biến, khó kiểm soát đâu! Sứ mệnh của nó là giúp chúng ta gom lại tất cả các xử lý tương tự nhau về làm một, viết code tối thiểu nhất có thể!
Kỹ thuật này là 1 cái gì đó rất trick/hacking nên khuyến cáo trước là làm rồi dễ ghiền lắm nhé! :))

Không phải tất cả các ngôn ngữ đều hỗ trợ metaprogramming, trong số những ngôn ngữ hỗ trợ thì cũng chỉ 1 số ít là được sử dụng rộng rãi trong thực tế.
Mình biết đến kỹ thuật này nhờ vào Ruby. Bởi vì nó khá là phổ biến ở đó, chúng ta có thể bắt gặp metaprogramming ở hầu hết các ruby repository trên Github!

See more at wikipedia
> https://en.wikipedia.org/wiki/Metaprogramming

## What Ruby has for metaprogramming?

Ngoài việc với thiết kế Metaclass - mọi thứ kể cả Class cũng thực chất là instance của 1 class cấp trên nào đó, 
theo mình yếu tố chính đưa metaprogramming trở lên phổ biến trong Ruby là nhờ vào các helper methods dưới đây.

- **instance_variable_set/instance_variable_get**

cho phép set/get biến instance dynamically

```ruby
  instance_variable_set("@foo#{'bar'}", 123)
# give us same result as
  @foobar = 123

  puts instance_variable_get("@foo#{'bar'}")
# give us same result as
  puts @foobar
```

- **send**

cho phép gọi method với tên bất kỳ

```ruby
  obj.send("foo#{'bar'}")
# give us same result as
  obj.foobar
```


- **define_method**

cho phép định nghĩa method dynamically

```ruby
  define_method "foo#{'bar'}" do
    # do something
  end
# give us same result as
  def foobar do
    # do something
  end
```

## Use metaprogramming when implement Rest APIs

### Before metaprogramming

Giả sử bạn cần viết Rest API cho 2 resource là Car và Bike. 
Làm như bình thường thì sẽ viết ra CarsController và BikesController kiểu như sau.

```ruby
class CarsController < Api::BaseController
  before_action :set_car, only: %i[show update destroy]

  def create
    ...
  end

  def show
    ...
  end

  def update
    ...
  end

  def destroy
    ...
  end

  private

  def set_car
    @car = Car.find(params[:id])
  end
end
```

```ruby
class BikesController < Api::BaseController
  before_action :set_bike, only: %i[show update destroy]

  def create
    ...
  end

  def show
    ...
  end

  def update
    ...
  end

  def destroy
    ...
  end

  private

  def set_bike
    @bike = Bike.find(params[:id])
  end
end
```

Mình không trích dẫn ra tất cả logic, nhưng ít nhất bạn cũng đã nhận ra phần logic tương tự bị lặp lại ở chỗ set_car/set_bike rồi chứ?
Tất cả chỉ là vì muốn tạo thêm 1 Rest API mới, nhưng mà class model cần tham chiếu đến thay đổi tùy từng API, và tên biến instance cũng cần thay đổi theo từng resource mà ta không thể làm class rồi kế thừa hay dùng kỹ thuật mixin để xử lý chung ở 1 chỗ được.

Trong công việc lập trình hàng ngày của chúng ta, có rất nhiều khi như vậy - chúng ta copy code từ class này qua class khác, rồi sau đó đổi tên biến, đổi tiên class vân vân đúng không?

### After metaprogramming

Giờ mình sẽ demo metaprogramming để gom tất cả những đoạn code có shared behavior đó lại.

Đầu tiên, trong mỗi controller của 1 Rest API, ta cần suy diễn ra tên resource đang cần xử lý là gì.
Ví dụ cars_controller.rb thì resource sẽ là car, và bikes_controller.rb thì là bike => có nghĩa là có thể lấy tên controller để suy ra tên resource!

```ruby
# Ex: cars_controller.rb will return "car" string
def resource_name
  @resource_name ||= controller_name.singularize
end

# Ex: cars_controller.rb will return "Car" class
def resource_class
  @resource_class ||= resource_name.classify.constantize
end
```

Có 2 methods trên rồi, tiếp theo phần truy cập database lấy ra dữ liệu của record tương ứng với id rồi set vào biến instance sẽ rất đơn giản. Mình sẽ sử dụng instance_variable_set/instance_variable_get ở đây.

```ruby
before_action :auto_set_resource, only: %i[show update destroy]

# Ex: cars_controller.rb will set up "@car" instance variable newly
def auto_set_resource(resource = nil)
  resource ||= resource_class.find_by!(id: params[:id])
  instance_variable_set("@#{resource_name}", resource)
end

# Ex: cars_controller.rb will return "@car" instance variable
def resource_object
  instance_variable_get("@#{resource_name}")
end
```

Như vậy chúng ta chỉ cần tạo controller, công việc còn lại như gọi đến model class, rồi set biến instance vân vân thì metaprograming sẽ dynamically thực hiện cho! Tất cả được tự điều chỉnh dựa theo tên của controller! Toẹt vời phải không.

Logic cơ bản cho 4 actions CRUD cũng có thể nhóm lại bằng metaprogramming theo cách tương tự. Các bạn xem chi tiết implementation ở đây nhé.
https://gist.github.com/chienkira/b6cb119912b2a90abbe34e8c4240d691

Thành quả cuối cùng là code controller của Rest API sẽ rút gọn được đến mức "cực tiểu" như dưới đây.
Mỗi khi cần thay đổi common behavior của toàn bộ Rest API, công việc cũng trở lên dễ dàng vì chỉ cần sửa duy nhất module RestfulResponsable.

```ruby
module Api
  module V1
    class CarsController < Api::V1::BaseController
      include ::RestfulResponsable

      # CRUD (create/show/update/destroy) logics are implemented inside RestfulResponable module

      private

      def car_params
        params.permit(:code, :maker, :number)
      end
    end
  end
end
```

---

Metaprogramming mang lại nhiều lợi ích như rút gọn code cho chúng ta, đóng băng các common behavior đảm bảo logic là duy nhất và thống nhất. Tuy nhiên đánh đổi lại, code readability sẽ bị giảm xuống chẳng hạn. Mình nghĩ áp dụng nó quá nhiều chưa chắc đã là ý tưởng hay, nên là chúng ta cũng nên chú ý đừng làm lố quá, kẻo bị thằng khác trong team nó chửi cho nha. :smiley:

**Happy metaprogramming!**