---
title: "Tìm Hiểu Về Api Gateway"
date: 2019-02-12T15:45:04+09:00
draft: no
tags: [aws, serverless, SAM, api gateway, lambda]
language: japanese
toc: true
---

# API GatewayとLambda間の処理フロー

APIGatewayとLambda間の処理フローは以下の図の通りです。
![LINK](https://docs.aws.amazon.com/apigateway/latest/developerguide/images/api-gateway-create-api-by-importing-example-post-method-execution.png) 

1. クライアントからHTTPリクエストがきた時に、[Method request] がそのリクエストを受け取って認証などを行う
2. API Gatewayは必要に応じて [Integration Request]を使い、リクエストのデータを変換してから、Lambdaに転送する
  ※ 変換されたデータがLambda関数のevent変数に入ります。
3. 次にAPI Gatewayは [Integration Response] を使い、Lambdaから返ってきた戻り値（処理結果）をまたデータ変換をしてから、[Method Response] に転送
4. 最後に、API Gatewayが [Method Response]にてクライアントに返信する 

 → **Integration Request** と **Integration Response**のデータ変換に関しては、以下の2選択肢があります。

* ①AWSのLambda proxy integration（Lambda プロキシ統合）に任せるか
* ②データ変換に使われるマッピングテンプレートを自分で定義するか

# Lambda proxy integration（Lambda プロキシ統合）

[AWSのLambda プロキシ統合の詳細説明](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html)
Lambda プロキシ統合を使用する場合、Lambda関数へのInputデータやLambda関数のOutputデータのフォーマットが決まってます。

## Lambda 関数の入力形式
以下のようにHTTPリクエスト全体をLambda 関数のevent変数にマッピングされます。
よく使うのは：

* headers: リクエストのヘッダデータ
* httpMethod: リクエストのメソッド情報
* pathParameters: GETリクエストのパスパラメータ
例）/hoge/{group}/{user}のようなリクエストの場合、pathParameters には、groupとuserが入ってきます。
* queryStringParameters: GETリクエストのクエリパラメータ
例）/hoge/{user}?page=5のようなリクエストの場合、queryStringParameters には、pageが入ってきます。
* body: POSTリクエストのPostデータ


```json
{
    "resource": "Resource path",
    "path": "Path parameter",
    "httpMethod": "Incoming request's method name"
    "headers": {String containing incoming request headers}
    "multiValueHeaders": {List of strings containing incoming request headers}
    "queryStringParameters": {query string parameters }
    "multiValueQueryStringParameters": {List of query string parameters}
    "pathParameters":  {path parameters}
    "stageVariables": {Applicable stage variables}
    "requestContext": {Request context, including authorizer-returned key-value pairs}
    "body": "A JSON string of the request payload."
    "isBase64Encoded": "A boolean flag to indicate if the applicable request payload is Base64-encode"
}
```
参照: (https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html#api-gateway-simple-proxy-for-lambda-input-format)

## Lambda 関数の出力形式
クライアントに正常に正しいHTTPステータスコードやレスポンスデータを返すために、以下の JSON 形式に従ってLambdaがデータを返却する必要があります。

```json
{
    "isBase64Encoded": true|false,
    "statusCode": httpStatusCode,
    "headers": { "headerName": "headerValue", ... },
    "multiValueHeaders": { "headerName": ["headerValue", "headerValue2", ...], ... },
    "body": "..."
}
```
参照: (https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html#api-gateway-simple-proxy-for-lambda-output-format)

メモ：
もしLambda プロキシ統合の方で決まっているフォーマットと違いデータをLambda関数が返した場合、
API Gatewayの中でエラーが発生し、クライアントに502 Bad Gateway エラーを返します。

# Mapping template
Lambda プロキシ統合を使用しない場合、どうAPI Gatewayにデータ変換してほしいか定義する必要がります。
その定義はMapping templateと言います。
データ変換は2箇所（Lambda関数のInputデータの変換とLambda関数のOutputデータの変換）があるため、
Mapping templateも [Integration Request] と [Integration Response] の2箇所で定義しなければなりません。

詳しいはこちらのリンクをご参考ください。
(https://dev.classmethod.jp/cloud/aws/api-gateway-mapping-template/)
(https://knackforge.com/blog/aws-api-gateway-request-response-mapping)

コードのサンプル（SAMのテンプレート式）
Mapping templateに書ける言語は [Apache VTL](https://qiita.com/yoshidasts/items/ddab0ef7b983b7c5260b)

```yaml
requestTemplates:
  application/json: |
    {
        #foreach($key in $input.params().querystring.keySet())
        "$key": "$util.escapeJavaScript($input.params().querystring.get($key))" #if($foreach.hasNext),#end
        #end
    }
```

```yaml
responseTemplates:
  application/json: |
    {
        $util.parseJson($input.json('$.body'))
        #set($Integer = 0)
        #set($status_code = $input.json("$.statusCode"))
        #set($status_code = $Integer.parseInt($status_code))
        #set($context.responseOverride.status = $status_code)
    }
```

# CORSについて
## CORS（Cross Origin Resource Sharing）とは
各ブラウザはクロスドメイン通信を拒否する仕組み（[same-origin policy](https://www.w3.org/Security/wiki/Same_Origin_Policy)）になっています。
その為、例えば`domain111.com`のサイトから`domain222.com`のAPIを叩いてデータ取得することが基本的にはできません。
しかし実際に他のドメインにAPIのアクセスを許可しなければならない時もあります。
その際は、CORSを設定すれば実現できます。

## ブラウザのCORSチェックについて
クライアント（ブラウザ）から送信するHTTPリクエストは以下の2週類で分類されます。

* Simpleリクエスト
リクエストのメソッドやContent-Typeなどの色々な条件に満たさないとSimpleリクエストと認められません。
実際にAPIを叩く時のリクエストはほとんどSimpleリクストではありません。
参考：(https://developer.mozilla.org/ja/docs/Web/HTTP/CORS#Simple_requests)
* Not Simpleリクエスト
Simpleリクエストの条件に満たさない全部のリクエストです。
参考：(https://developer.mozilla.org/ja/docs/Web/HTTP/CORS#Preflighted_requests)

リクエストの種類によって、ブラウザのCORSチェックが異なります。

* Simpleリクエストの場合
ブラウザがリクエストのレスポンスHeadersを見て、以下のHeaderをチェックします。
このHeaderがない場合又は、Headerの値にオリジンドメインが入っていない場合は
クロスドメイン通信が拒否されます。

```
Access-Control-Allow-Origin: http://api.bob.com
```

* Not Simpleリクエストの場合
対象のリクエストを送信する前に、ブラウザが勝手にCORSチェックのリクスト（pre-flightリクスト）を送信します。
pre-flightリクエストのレスポンスを確認した上で、通信可能な場合のみ対象のリクエストを送信するという順番です。
※ pre-flightリクエストはOPTIONSメソッドで送信されます。

```
Access-Control-Allow-Origin: http://api.bob.com
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: X-Custom-Header
```

## CORS設定方法（SAMの場合）
* Headerの対応
Lambda関数の処理の方で、以下のHeaderを加えて返す必要があります。

```python
headers = {
            # CORS support
            'Access-Control-Allow-Origin': '*',
        }
```

* Pre-flightリクエストの対応
SAMの場合は以下のようにtemplate.ymlよりAPI Gatewayの記述で簡単に対応できます。

```yaml
HogeApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: Dev
      Cors:
        AllowMethods: "'*'"
        AllowHeaders: "'*'"
        AllowOrigin: "'*'"
      Auth:
<省略>
```

 **注意点：** 
API Gatewayにカスタムオーソライザの設定があった場合、
以下のようにDefaultAuthorizerを記述するとpre-flightリクスト対応のOPTIONSメソッドにも認証が付いてしまう為、
pre-flightが通らなくなります。

```yaml
Auth:
        DefaultAuthorizer: MyTokenAuthorizer
        Authorizers:
          MyTokenAuthorizer:
<省略>
```

→対応方法：
DefaultAuthorizerの記述を消して、各Lambda関数のところにAuthorizerを指定するようにすれば、
認証がOPTIONSメソッドから外されます。

```yaml
HogeFunction:
    Type: AWS::Serverless::Function
    Properties:
      <省略>
      Events:
        GetResource:
          Type: Api
          Properties:
            <省略>
            Auth:
              Authorizer: MyTokenAuthorizer
```
