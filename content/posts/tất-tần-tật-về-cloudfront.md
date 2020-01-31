---
title: "Understand AWS Cloudfront (日本語)"
date: 2019-02-21T16:33:07+09:00
draft: no
tags: [aws, cloudfront]
language: japanese
toc: true
authors: [chienkira]
---

# CloudFrontとは

- AWSが提供するCDN（コンテンツ配信サービス）である。
- 世界中に設置されているエッジサーバを利用し、ユーザの最寄りエッジサーバからキャッシュを送ることでユーザへ高速な配信を実現できるサービスである。

![CloudFrontのコンテンツをユーザーに配信する方法]
(https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/images/how-cloudfront-delivers-content.png)

# CloudFrontの設定

## Distribution Settings（ディストリビューション設定）
ディストリビューションはCloudFrontの配信設定の単位になるので、CloudFrontの使用にはまずディストリビューションを作成する必要がある。

ディストリビューション配信種類はWebとRTMPがある。
- Web: ウェブ配信専用（基本的にこちらの種類を使う）
- RTMP: メディアのストリーミング配信専用

1. Alternate Domain Names (CNAMEs)
CloudFront によって割り当てられたドメイン名（例 `https://hogehoge.cloudfront.net`）の代わりに、使用したい代替ドメイン名（例 `https://example.com`）をここに指定する。

2. SSL Certificate
デフォルトのCloudFrontのSSL証明書を使用するか、独自SSL証明書（ACMで登録したもの）を選択する。


## Origin Settings（オリジンドメイン設定）
1. Origin Domain Name  
オリジンドメインを選択する。
フロントエンドをS3バケットに格納した場合は、オリジンドメインにS3バケットを選択すると良い。  

    ※オリジンドメインとは、コンテンツの提供元のことを表す。オリジンドメインには、AWSのリソース（S3バケットやELBやAWS MediaPackageエンドポイントや AWS MediaStoreContainerエンドポイント）か、もしくはそれ以外のリソース（どこかのウェブサーバのドメイン等）でも指定できる。

2. Origin Path
オリジン内のディレクトリからコンテンツが配信されるようにしたい場合は、オリジンパスを入力することで実現できる。
例えば、オリジンパスに`/green`を入力した場合、ユーザーがブラウザで`example.com/index.html`とリクエストすると、CloudFrontは `s3bucket/green/index.html` を返答する。

3. Origin ID
ディストリビューション内でこのオリジンをユニークで区別する為の文字列である。
オリジンドメインを入力したら、自動的に生成されるオリジンIDをそのまま使っても良い。

4. Restrict Bucket Access
CloudFrontのURLでしかS3バケット内のコンテンツがアクセスできないように制御をしたい場合、この設定をYESにする。
ユーザーがCloudFrontのURLでもS3バケットのURLでもアクセスできるようにしたい場合は、[No]にする。

5. Origin Access Identity
[Restrict Bucket Access] で [Yes] にした場合、オリジンアクセスアイデンティティが必要になる。
新しいアイデンティティを作成するか、既存のアイデンティティを使用するかを下の [Your Identities] 設定項目で選択する。

6. Grant Read Permissions on Bucket
CloudFront で、S3 バケット内のオブジェクトの読み取りアクセス許可を自動的に付与するには、[Yes, Update Bucket Policy] を選択する。
アクセス許可を手動で更新する場合は、[No, I will Update Permissions] を選択する。

7. Origin Custom Headers 
CloudFront がオリジンにリクエストを転送するときに、常にプラスのカスタムヘッダーを含めるように設定する。


## Cache Behavior Settings（キャッシュ動作設定）
キャッシュ動作設定を使用すると、URLパスパターンに対して、様々な機能が設定できる。

1. Path Pattern
このキャッシュ動作をどのようなリクエストに割り当てるかを指定する。
パスパターンの例：`images/*` や `*.gif` など
デフォルトは `*` (すべてのファイル) に設定される。
パスパターンには、以下のワイルドカード文字を使用できる。
`* は、0 個以上の文字に一致`
`? は、正確に 1 個の文字に一致`

2. Origin or Origin Group
オリジン（ユニークなオリジンID）を選択する。
例えば、複数のオリジンを設定している場合に、パスパターンで判断しリクエストを違うオリジンに転送することができる。

3. Viewer Protocol Policy
[HTTP and HTTPS]: ユーザはHTTPでもHTTPSでもコンテンツアクセスを許可する。
[Redirect HTTP to HTTPS]: ユーザがHTTPでアクセスした場合、自動的に HTTPS リクエストにリダイレクトさせる。
[HTTPS Only]: ユーザはHTTPSでしかコンテンツアクセスできないようにする。

4. Allowed HTTP Methods
CloudFrontが処理してオリジンに転送可能なHTTPメソッドを設定する。

5. Lambda Function Associations
CloudFront が配信するコンテンツをカスタマイズするLambda @Edge関数を実行できる。
(https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/lambda-at-the-edge.html)
Lambda @Edge関数は、次の4つのCloudFrontイベントの発生時に実行することが可能である。
    - Viewer Request：CloudFront がユーザからのリクエストを受信したとき
    - Origin Request：CloudFront がリクエストをオリジンに転送する前 
    - Origin Response：CloudFront がオリジンからレスポンスを受信したとき
    - Viewer Response：CloudFront がユーザにレスポンスを返す前

    Lambda @Edgeを使用すれば、様々なことが実現可能になる。
    例）
    ・Origin Requestのトリガーを使い、Cookie を検査して異なるオリジン（S3バケット等）からコンテンツを配信させることができる。
    ・デバイスに関する情報が含まれている User-Agent ヘッダーを確認し、異なるコンテンツ（画像など）をユーザに返すことができる。
    ・等々
    なお、普通のLambda関数と違う実行環境や制御がある。詳細はこちらのリンクご参照ください。
    (https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/lambda-requirements-limits.html#lambda-requirements-lambda-function-configuration)

## Invalidation Settings（キャッシュ無効化設定）
有効期限切れになる前にCloudFrontのエッジサーバから、キャッシュを削除したい場合、この「キャッシュの無効化」設定をする。
※キャッシュが無効化された後、エッジサーバにリクエストがあるとエッジサーバがオリジンから最新コンテンツを取得して配信する。

1. Distribution ID（参照のみ）
対象のディストリビューションのIDが表示される。

2. Object Paths
無効化したファイルのパス+ファイル名を入力する。
ワイルドカードを使用することで（`/*`）一括指定することも可能である。


## Custom Error Response Settings（カスタムエラーレスポンス設定）
オリジンが HTTP 4xx または 5xx のステータスコードを CloudFront に返した場合に、ユーザにカスタムエラーページ (HTML ファイルなど) を返すことができる。
1. HTTP Error Code
オリジンから返ってきたステータスコード（ドロップダウンから4xxと5xxのエラーが選択可能）

2. Error Caching Minimum TTL (seconds)
エラーレスポンスをキャッシュする最小時間（秒）
※この時間が過ぎていない限りにはオリジンに新しくリクエストを送らず、キャッシュされたエラーレスポンスを使ってユーザに返す。

3. Response Page Path
CloudFront がユーザに返すカスタムエラーページのパス (例: /errors/403-forbidden.html)

4. HTTP Response Code
カスタムエラーページとともにユーザに返す HTTP ステータスコード


## Geo-Restriction Settings（地域制限設定）
特定の国のユーザーに対して、コンテンツのアクセスを許可するしないことがでこいる。


# CloudFrontの料金

## CloudFrontの料金
合計料金 = インターネットへのデータ転送アウト料金 + オリジンへのデータ転送アウト料金 + リクエスト料金

各料金はリージョン別で異なる。
以下はJapanリージョンの場合の料金情報となる。

- インターネットへのデータ転送アウト料金
  月の転送データ量によって、1 GBあたりの料金が異なる。

転送データ量 | 1GBあたりの料 
--- | ---
  月10 TB まで   | 0.114 USD
  月10 TB 超 50 TB まで   | 0.089 USD	
  月50 TB 超 150 TB まで  | 0.086 USD	
  月150 TB 超 500 TB まで   | 0.084 USD	
  月500 TB 超 1,024 TB まで   | 0.080 USD	
  月1 PB 超 5 PB まで   | 0.070 USD	
  月 5 PB 超  | 0.060 USD	

- オリジンへのデータ転送アウト料金 
  1 GBあたり0.060USD
- リクエスト料金
  HTTP リクエスト1 万件あたり0.0090 USD
  HTTPS リクエスト1 万件あたり0.0125 USD

## Lambda @Edgeの料金
合計料金 = コンピューティング料金 ＋ リクエスト料金

- コンピューティング料金
  128MB-秒 ごとに 0.00000625125 USD
- リクエスト料金
  リクエスト 100 万件あたり 0.60 USD
