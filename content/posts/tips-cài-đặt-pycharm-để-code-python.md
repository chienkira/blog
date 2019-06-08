---
title: "Tips Cài Đặt Pycharm Để Code Python(日本語)"
date: 2019-02-08T15:49:01+09:00
draft: no
tags: [python, pycharm]
language: japanese
authors: [chienkira]
---

### PycharmのInterpreterの設定方法
python仮想環境を使い、開発する際にはPycharmがそのpython仮想環境を認識させる必要があります。  
※ 認識させないと、Pycharmの方がインストールされているライブラリが分からずimportのエラーなどが出てしまいます。  

- ① Python仮想環境の場所を確認    
  pipenvの場合はこのコマンドで確認できます。 `pipenv --venv`  
  output例： `/Users/kira/.local/share/virtualenvs/project_name-Qr43IEm2`  
- ② Pycharmの方でプロジェクトのInterpreterを設定  
  * PycharmのPreferences画面を開く  
  * 左側のメニューから「Project: プロジェクト名」=>「Project Interpreter」を選択  
  * Project Interpreterのドロップダウンから「Show All」を選択し、「+」の追加ボタンを押す  
  * 左側のメニューから「System Interpreter」を選択し、「...」ボタンをクリックして①で確認できたパスを指定  
  ※ 注意：bin/pythonX.Xまで指定してください。

### PycharmのSources Root設定方法
開発ソースコードがプロジェクトの直下にはなく、srcフォルダーなどの中に置かれる場合が多いです。  
※ CodeUriが./srcで設定される場合が多いです。 

例えば以下のようなファイル構成の場合
```bash
|--template.yaml
|--src
|  |--app
|  |  |--__init__.py
|  |  |--common
|  |  |  |--__init__.py
|  |  |  |--helper.py
|  |  |--handlers
|  |  |  |--__init__.py
|  |  |  |--purchase.py
```
purchase.pyファイルで`app/common/helper.py`のモジュールなどをimportするところの`from app.common.helper import some_helper`
にPycharmがエラーを表示します。  

エラー解消にはソースコードのルートフォルダーをsrcでPycharmに認識させる必要があります。
- Pycharmでsrcフォルダーを右クリック
- Mark Directory asの「Sources Root」を選択
