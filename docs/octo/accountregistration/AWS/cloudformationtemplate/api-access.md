!!!note "ℹ️ 以下のドキュメントは機械翻訳により提供されています。提供された翻訳内容に関して、不明点または矛盾がある場合、英語版の方を参照ください。"

# OctoにおけるAPIアクセスのセットアップ

このCloudFormationテンプレートは、**Alphaus Octo**が以下の目的でAWSアカウント内のリソースへ安全にアクセスするために必要なIAMロールをプロビジョニング（作成）します：

- コスト最適化の推奨
- 変更実行の自動化
- コストと使用状況のレポート
- Trusted AdvisorおよびCompute Optimizerの分析
- SSM Automation（オートメーション）の統合
- エンドツーエンドのコスト最適化ワークフロー

このテンプレートは、**AWSアカウント**へのデプロイを意図しています。

---

# 1. パラメータ (Parameters)

## 1.1 `Principal`
- **Type (型):** String
- **Description (説明):** ロールの引き受け（Assume Role）を許可されたAlphausのAWSアカウントID。
- **Pattern (パターン):** 数字のみ (`^[0-9]*$`)

## 1.2 `ExternalId`
- **Type (型):** String
- **Description (説明):** クロスアカウントセキュリティのために `sts:AssumeRole` 実行時に使用される外部識別子。

両方のパラメータは、「*混乱した代理人（confused deputy）問題*」から保護するためのものです。

---

# 2. 作成されるIAMロール (IAM Roles Created)

このテンプレートは、それぞれ特定の目的を持つ**5つのIAMロール**を作成します。

---

# 2.1 OctoRecommendationSearcherRole

### 目的 (Purpose)
Alphausがコストおよび最適化の推奨に必要な**読み取り専用のリソース情報**を収集できるようにします。

### 機能 (Capabilities)
- Compute Optimizerデータ
- Trusted Advisorチェック
- CloudWatchメトリクス
- EC2, EBS, RDS, Lambda, Redshift, DynamoDBなどのDescribe/List（説明/一覧表示）
- CloudTrailルックアップ
- S3バケットメタデータ
- Route53ホストゾーン
- 料金データ
- Organizationsメタデータ
- Backupデータ
- Resource Groups情報

### セキュリティ (Security)
- **完全な読み取り専用**
- 変更や削除のアクションは不可

---

# 2.2 OctoChangeExecutorRole

### 目的 (Purpose)
AWSリソースへの変更を実行する、**承認されたSSM Change Managerワークフロー**を実行します。

### 引き受け元 (Assumed By)
- `ssm.amazonaws.com`
- そのAWSアカウントのルートプリンシパル

### 機能 (Capabilities)
- AWS管理ポリシー：**AdministratorAccess**
- 顧客によってトリガーされたSSMオートメーションランブックを実行

### セキュリティ (Security)
- **Alphausがこのロールを直接引き受けることはできません。**
- 顧客のSSMワークフローのみがこれを使用できます。

---

# 2.3 OctoSSMUpdaterRole

### 目的 (Purpose)
Alphausが**SSMオートメーションドキュメントを管理**し、変更リクエストを開始できるようにします。

### 引き受け元 (Assumed By)
- `Principal` + `ExternalId` を使用したAlphausアカウント

### 機能 (Capabilities)
- SSMドキュメントの管理（作成/更新/削除）
- オートメーション実行のトリガー
- OctoChangeExecutorRoleへのPassRole（ロールの受け渡し）
- タグ付け、サービス設定の更新
- SSMオートメーションの停止、再開、クエリ

### セキュリティ (Security)
- AWSリソースを直接変更することはできません
- すべての変更は依然として SSM + OctoChangeExecutorRole を経由します

---

# 2.4 OctoChangeTemplateApproverRole

### 目的 (Purpose)
Change Managerテンプレートに対する**顧客側の承認**を可能にします。

### 引き受け元 (Assumed By)
- 顧客のAWSアカウントのみ

### 機能 (Capabilities)
- SSMドキュメントの閲覧
- 変更テンプレートの承認、改訂、または提出
- オートメーション実行シグナルの提供

### セキュリティ (Security)
- Alphausはこのロールへのアクセス権を**持ちません**。

---

# 2.5 AlphausAcctAccessRole

### 目的 (Purpose)
Alphausがアカウントレベルのコストと使用状況のメタデータを読み取る機能を提供します。

### 機能 (Capabilities)
- Cost Explorer (`ce:*`)
- CUR (`cur:Describe*`)
- Savings Planおよびリザーブドインスタンスの推奨
- CloudWatchメトリクス
- RIインベントリ確認のためのRDS, Redshift, ElastiCacheへのDescribeアクセス
- AWS Organizationメタデータ
- IAM ListRoles
- CloudFormation読み取り専用アクセス

### 追加のIAM権限 (Additional IAM Permissions)
- 以下のインラインポリシーを更新可能：
  - `AlphausAcctAccessRole`
  - `OctoRecommendationSearcherRole`
- **このロールを作成した**スタックの更新が可能（テンプレートのアップグレードに使用）
- 以下に必要なサービスリンクロールの作成が可能：
  - Trusted Advisor
  - Cost Optimization Hub

---

# 3. セキュリティアーキテクチャ (Security Architecture)

### クロスアカウントアクセスの保護
- `Principal`（Alphaus AWSアカウントID）を使用
- `ExternalId`を使用
- ロールは範囲を限定した最小限の権限を持つ
- 不要な書き込み機能を持たせない

### ロールによる職務分掌

| ロール | アクセスレベル | 目的 |
|------|--------------|---------|
| OctoRecommendationSearcherRole | 読み取り専用 | 最適化データの収集 |
| OctoSSMUpdaterRole | SSM書き込み | オートメーションドキュメントの管理 |
| OctoChangeExecutorRole | 管理者 (Admin) | 承認された変更の実行 |
| OctoChangeTemplateApproverRole | 承認 | ワークフローの承認 |
| AlphausAcctAccessRole | 読み取り専用 | 請求、CUR、CE、およびメタデータへのアクセス |

これにより権限昇格を防ぎ、顧客による制御を保証します。

---

# 4. デプロイに関する注記 (Deployment Notes)

### 推奨されるデプロイ方法
- **AWS CloudFormation StackSets**
  - すべての**メンバーアカウント**へデプロイ
  - 必要に応じて管理アカウントを除外
  - CURが有効になっていないアカウントを除外

### アップデートの安全性
ロールは、自身を作成したCloudFormationスタックの更新が許可されており、以下を保証します：
- 既存環境を破壊しないテンプレートのアップグレード
- 自動化されたポリシー更新

---

# 5. まとめ (Summary)

このCloudFormationテンプレートは以下を可能にします：

| 機能 | 担当するロール |
|------------|------------------|
| リソースデータの収集 | OctoRecommendationSearcherRole |
| 変更ワークフローの実行 | OctoChangeExecutorRole |
| SSMオートメーションの管理 | OctoSSMUpdaterRole |
| 顧客によるワークフロー承認 | OctoChangeTemplateApproverRole |
| コスト、請求、CUR、CEへのアクセス | AlphausAcctAccessRole |
| 安全なクロスアカウント認証 | Principal + ExternalId |

このテンプレートは、**Alphaus Octo**がコスト最適化の推奨と自動化を行うために必要なすべての権限を提供しつつ、**強力な顧客側の制御とセキュリティ境界**を維持します。