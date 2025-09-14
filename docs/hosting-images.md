# S3画像のホスティング

S3から直接ホスティングせず、Cloudfrontを通じて配信するようにしましょう。

## 全体設計

```mermaid
flowchart TB
  subgraph UserSide[ユーザー側]
    B[ブラウザ/SPA]
  end

  subgraph AWS[あなたのAWSアカウント]
    WAF[WAF（ウェブアプリケーションファイアウォール）]:::edge
    CF[CloudFront（CDN：世界分散キャッシュ）]:::edge
    OAC[OAC（オリジンアクセスコントロール）]:::ctrl
    S3[(S3バケット\n・originals/ に原本\n・derived/ にサムネ)]:::store
    L[Lambda（サムネ生成）]:::compute
    R[Rekognition（モデレーション：任意）]:::ai
    VPCE[Interface VPCエンドポイント（管理用：任意）]:::net
    IDP[Cognito Identity Pool（連携IDプール：一時認証）/STS]:::auth
  end

  %% 閲覧フロー（閲覧者→CloudFront→S3）
  B -- "画像閲覧（HTTP→HTTPSへリダイレクト）" --> WAF --> CF
  CF -- "S3読み取り\n（署名付き接続）" --> OAC --> S3

  %% アップロードフロー（投稿者→S3 PUT）
  B -- "認証→一時認証情報の取得" --> IDP
  B -- "PUT オブジェクト\n(originals/{userId}/{uuid}.ext)" --> S3

  %% 生成フロー（S3イベント）
  S3 -- "ObjectCreated イベント" --> L
  L -- "サムネ生成\n(例: w192/webp など)" --> S3

  %% 任意の追加
  S3 -- "モデレーション（任意）" --> R
  VPCE -. "社内/管理からのS3私設ルート（任意）" .- S3

  classDef edge fill:#eef,stroke:#88a,stroke-width:1px;
  classDef ctrl fill:#ffd,stroke:#cc9;
  classDef store fill:#efe,stroke:#7a7;
  classDef compute fill:#fee,stroke:#c88;
  classDef ai fill:#f5e,stroke:#a4a;
  classDef auth fill:#def,stroke:#59c;
  classDef net fill:#eee,stroke:#bbb;
```

## 画像の読み出し

決まったパスをCloudfrontからPublicにホスティングするため、どこからでもHTTPS経由で表示できます。

```mermaid
flowchart LR
	S3 --> CloudFront
	CloudFront --> HTTPS
```

## 画像の登録

画像がアプリ側から直接S3に登録します。

API Gateway経由にすることもできますが、制限があるため AWS SDK を使用して直接S3にアップロードする方法が一般的です。

この時、アプリ側に直接IAM情報をハードコートすると危険なので、必ず一時認証の Cognito ID Pool に紐づけたIAMロール（AuthRoleまたはUnauthRole）に権限をアタッチするようにしましょう。

```mermaid
flowchart LR
	User -- "認証" --> CognitoIdp[Cognito ID Pool]
	CognitoIdp -- "AuthRole または UnauthRole" --> S3
```

### 画像の加工

デバイス別、サムネイル用など１つの画像から複数サイズの画像もS3トリガーを使用してLambdaで自動生成することができます。
