---
title: "\"Thông\" Mạng Cho Aws SAM Và Dynamodb Local(日本語)"
date: 2019-02-13T15:50:07+09:00
draft: no
tags: [aws, serverless, SAM, dynamodb local, docker]
language: japanese
authors: [chienkira]
---

### Dynamodb localを使いローカルで開発時
SAM localのstart-apiやinvokeコマンドを使い、Lambda関数をローカルで実行する時は、  
- Lambda関数がSAMのDockerコンテナ上で実行され
- Dynamodb localが別のDockerコンテナ上で動く

そのため、Lambda関数がDynamodb localへアクセルできるように、
2つのDockerコンテナを同じネットワークに繋がせる必要があります。

対応方法は以下の通りです。

まずはDynamodb localのDockerセットアップ `file: docker-compose.yml`
```yaml　
services:
  dynamodb:
    container_name: dynamodb  #重要：コンテナ名を指定
    image: amazon/dynamodb-local
    networks:
      - aws_local_network  #繋がるネットワークを指定
   <省略>

networks:
  aws_local_network:
    name: aws_local_network  #重要：ネットワーク名を強制的に指定
```
そして、モデルのソースコードを以下のように修正
```python
if os.environ.get('AWS_SESSION_TOKEN') is None:
    host = "http://dynamodb:8000"
```
　　解説： [AWS_SESSION_TOKEN](https://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html) 環境変数には、実際のAWS環境で実行する時にしか値がないので  
　　ローカルで実行しているかどうなの判別に使えます。  
　　また、`http://dynamodb:8000`の`dynamodb`はDynamodb localのコンテナ名です。

最後に、Lambda関数をローカルで実行する時 `--docker-network aws_local_network` を追加で指定  
`sam local start-api --docker-network aws_local_network`  
又は、`sam local invoke HogeFunction -e event.json --docker-network aws_local_network`  
