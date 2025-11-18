!!!note "ℹ️ 以下のドキュメントは機械翻訳により提供されています。提供された翻訳内容に関して、不明点または矛盾がある場合、英語版の方を参照ください。"

# OctoでのCURおよびS3バケットの設定

このテンプレートは、AWS Cost and Usage Report（CUR） をS3へエクスポートする設定を行い、AlphausがエクスポートされたCURファイルおよび関連する請求データに安全にアクセスできるようにするIAMロールをプロビジョニングします。

## テンプレート情報
Template Information

- Format Version: 2010-09-09
- Description: CURエクスポートおよび、Alphausが貴社アカウントのコスト関連情報にアクセスするためのクロスアカウントIAMロールを設定します。

## パラメーター

このテンプレートでは、デプロイ時に以下の2つのパラメーターが必要です：

| パラメーター          | 種別     | 説明                                                     | デフォルト            | 備考                                      |
| ------------------ | -------- | --------------------------------------------------------------- | ------------------ | ------------------------------------------ |
| CurS3BucketOption  | String   | 新しいS3バケットを作成するか、既存のバケットを使用するかを選択します。
| CREATE_NEW         | 許可される値: CREATE_NEW, USE_EXISTING.  |
| CurS3BucketName    | String   | CURを保存するS3バケット名です。                             | alphaus-cur-export | USE_EXISTINGを選択した場合は、バケットが既に存在している必要があります。      |
| CurS3BucketRegion  | String   | S3バケットのリージョンを指定します。                                       | us-east-1          | 新しいバケットを作成する場合はこの値が使用されます。       |
| CurS3Prefix        | String   | CURレポートのフォルダープレフィックスです。                                  | pre                | Nスペースは使用できません                         |
| CurReportName      | String   | CURレポートの名前です。                                            | curreport          | 一意である必要があり、大文字小文字を区別し、スペースを含めないでください。 |
| Principal          | String   | CURデータにアクセスするAlphausのAWSアカウントIDです。           | —                  | 数値のみ使用可能です。      

## 条件

- CreateBucket – CurS3BucketOption = CREATE_NEW の場合に true となり、S3バケットの作成がトリガーされます。

## リソース

S3Bucket（任意）
- タイプ：AWS::S3::Bucket
- CurS3BucketOption = CREATE_NEW の場合にのみ作成されます。
- エクスポートされたCURファイルを保存します。

S3BucketPolicy
- billingreports.amazonaws.com サービスに以下のアクセス権限を付与します：
    - バケットのACLおよびポリシーの読み取り（s3:GetBucketAcl, s3:GetBucketPolicy）
    - CURファイルをバケットに書き込む権限（s3:PutObject）

CUR Definition (CurDef)

- タイプ：AWS::CUR::ReportDefinition
- CURエクスポートのプロパティを定義します：
    - 形式：textORcsv
    - 圧縮：ZIP
    - 粒度：HOURLY（1時間ごと）
    - 追加データ：リソースレベルの詳細を含みます（RESOURCES）
    - バージョニング：OVERWRITE_REPORT（新しいレポートが古いレポートを上書きします）
    - 指定されたバケットおよびプレフィックスにエクスポートされます。

IAMロール（IAM Role）：AlphausAcctAccessCurRole
- タイプ（Type）：AWS::IAM::Role
- ロール名：AlphausAcctAccessCurRole
- 信頼ポリシー：
    - Principal パラメーターで指定された AlphausのAWSアカウント によって信頼されます。
    - 最大セッション時間：12時間（43,200秒）

アタッチされたポリシー（インライン）

1. S3アクセス
    - CURバケットおよびオブジェクトに対する s3:Get, s3:List 権限。

2. コストおよび請求データへのアクセス
    - organizations:List, organizations:Describe
    - ec2:DescribeReservedInstances, rds:DescribeReservedDBInstances, elasticache:DescribeReservedCacheNodes, es:DescribeReservedElasticsearchInstances, redshift:DescribeReservedNodes
    - savingsplans:Describe*
    - cur:Describe, budgets:Describe
    - ce:Get, ce:List, ce:Describe*
    - billingconductor:List*

3. IAMロール管理（スコープ付き）
    - iam:GetRole*, iam:PutRolePolicy restricted to AlphausAcctAccessCurRole.
4. CloudFormation統合
    - cloudformation:DescribeStack, cloudformation:UpdateStack limited to the stack itself.

## セキュリティに関する考慮事項

- スコープ付きアクセス：
    - S3の権限は、指定されたバケットおよびオブジェクトのみに制限されています。
    - IAMアクションも AlphausAcctAccessCurRole に限定されています。
- 削除権限なし：
    - APIアクセス用のロールテンプレートとは異なり、このロールには EC2 や RDS などの削除権限はありません。（e.g., delete EC2, RDS）
- 外部アクセス制御：
    - Alphausのアクセスは、Principal AWSアカウントID によって制御されています。