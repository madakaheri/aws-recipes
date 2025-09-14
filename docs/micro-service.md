# マイクロサービス

マイクロサービスの構築は非常にシンプルですが、私は API Gateway は接続せずサーバー・クライアント形式のSDKと併せてサービスを構築する方法を勧めています。

## Microservice Framework

こちら **[microservice-framework](https://github.com/madakaheri/microservice-framework)** を使用すればある程度楽に作成できると思います。

## 全体設計

```mermaid
flowchart LR
	SDK["Service SDK（GitHub Packages）"] --> ServiceGateway["Service Gateway（Lambda）"]
	ServiceGateway --> SpecialFunction["Special Function（Lambdaなど）"]
```

### Service SDK

プライベートパッケージとして配布するサービス・クライアントです。  
内部には各種アクションメソッドがあり、それぞれが Service Gateway に対して Lambda:invoke() を行うだけのシンプルな構造です。

### Service Gateway

Service Gateway はその名の通り、内部で複数のアクションをルーティングして実行し、結果を返します。

### Special Function

Service Gateway から別の Lambda などを実行する際は、Service Gatewayに実行権限をアタッチする必要があります。

## この構造にする理由

### 安易に API Gateway を生やす文化にしたくない

チームで制作しているとPostmanなどから実行確認したい人が安易にPublicなAPIを生やしがちです。  
そしてそのAPIが付けられたテンプレートで本番デプロイまでしてしまいます。  
私も多少多めに見ていますが、よろしくないのは確かです..

それはいいとして、API Gatewayにしてしまうと実行権限フリーにしがちなので、私はクライアント側が明示的に Lambda:invoke 権限を設定しないと使えないようにこの方式を勧めています。

あとは API Gateway の制限がキツすぎることが挙げられます。

## Service Gatewayに実行権限を集約できる

クライアントがサービスAPIを使用する際に複数のLambda実行権限とDynamo、S3など多くの権限をアタッチしないと使えないようでは非常に煩雑です。  
こちらの構成であれば、クライアント側は Service Gateway の Lambda:invoke 権限だけをアタッチすれば使用できのでシンプルになります。
