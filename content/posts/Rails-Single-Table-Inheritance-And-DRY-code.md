---
title: "Rails Single Table Inheritance and DRY Code (日本語)"
date: 2020-01-24T14:46:24+09:00
draft: false
tags: [rails, DRY, programing, ruby, design]
language: vi
toc: true
authors: [chienkira]
---

**STI（Single Table Inheritance/単一テーブル継承）をRailsでどう実装するかについて、情報をまとめて残したいと思います。**

## STIの概要

- 英語：Single Table Inheritance
- 日本語：単一テーブル継承

OOPプログラミングの世界でよく知られている「継承」技術の一つです。

共通項目、共通振る舞いを折り出して**スーパークラス**を作り、
非共通項目や振る舞いはそのスーパークラスの**継承したサブクラス**で定義するというやり方ですね。

こうすることで、コードの重複を防げます。メンテナンスもやりやすいコードになります。

データベースの設計にも適用しようというのがSTIです。
特に、モデル間の共通データがほとんどだが、振る舞いが異なる場合はSTIが一番適切です。

## RailsでSTIを実装してみる

### 例のシステム

- スマホを販売している会社がスマホのデータを管理したい
- スマホの種類は、iPhoneとAndroidの2種類がある
- 種類に関係なく、製造番号やメモリや画面サイズなどのデータが共通となる（**→ STIに適切ヒントですね**）
- なお、スマホの種類によって異なるアクションができるようにする必要がある。
  例えば、スマホの最新OSバージョンチェック機能があるとして、
  - iPhoneの場合AppleのAPIを叩く必要がある、
  - 対して、Androidの場合GoogleのAPIを叩かないといけない

### Rails実装

何にも気にしないで作ると、iphone/androidそれぞれのテーブル、モデルを作成しますよね。
そして、共通のデータやメソッドもあるので、
似たような記述・コピペが増えそうな予感を感じますよね！

それを避ける為に、STIを適用してみしょう。
↓の図をご覧ください。

![STI Rails](/blog/images/sti_rails.svg)

一見どうということもないように見えますが、実は

- iPhoneとAndroidテーブルは**存在しない**
- 全てのスマホのデータが、**smartphoneテーブル一つだけ**に保存される

スーパーテーブルを用意して、継承のサブテーブルを作成するということです。

データレベルでは、スマホのデータしか実在しませんが、
アプリケーションレベル（Rails上）では、スマホを継承したiPhoneとAndroidモデルが存在します。

#### Railsで実際に実装する

###### 1. マイグレーションを作成

スマホのテーブルしか存在しないので、マイグレーションファイルは1個になります。

```ruby
class CreateSmartphoneTables < ActiveRecord::Migration[5.2]
  def change
    create_table :smartphones do |t|
      t.string :type, null: false
      t.string :serial_no
      t.float :screen_size
      t.integer :memory
      t.timestamps null: false
    end
  end
end
```

ポイントは、`type`カラムを必須で用意する必要があることです。
RailsにSTIを使っているよって伝えるためです。

後で、このテーブルの実データを確認してみると、
 `type` カラムに `Smartphone::iPhone` か `Smartphone::Android` の値が入ってることが分かります。

###### 2. モデルクラスを実装

次に、モデルクラスを実装しましょう。

最初は、Smartphoneモデルですね。通常の形で実装するだけです。

```ruby
class Smartphone < ApplicationRecord
  # 共通バリデーション
  validates :serial_no, presence: true

  # 共通メソッド
  def say_hi
    puts "Hi"
  end

  # 後で定義する
  def latest_os_version
    raise NotOverrideError
  end
end
```

- 共通のバリデーションやメソッドはここで定義しましょう。
- また、先ほど説明した `type` カラムは、もし別のカラムを使いたい場合はこれで変えられます。

```ruby
class Smartphone < ApplicationRecord
  self.inheritance_column = :smartphone_class # ここでinheritance_columnのカラムを指定
end
```

---

そして、やっとIphoneとAndroidモデルを実装してみましょう。

```ruby
class Smartphone
  class Iphone < Smartphone
    # 再定義する
    def latest_os_version
      call_APPLE_api :apple:
      parse_response
      ...
    end
  end
end
```

```ruby
class Smartphone
  class Android < Smartphone
    # 再定義する
    def latest_os_version
      call_GOOGLE_api :thinking:
      parse_response
      ...
    end
  end
end
```

気をつけないといけないのは、IphoneやAndroidクラスが継承するのはApplicationRecordでなく親であるSmartphoneですね。

これで、親で設定したリレーションやバリデーションやメソッドが全部子クラスにも引き継がれます。
なお、latest_os_versionメソッドの振る舞いはそれぞれ異なって実装することができますね！

**DRY（重複排除）コードを目指すなら、適用できる場合はSTIを使いましょう。**
