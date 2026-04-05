---
title: "Claude Code 本番DB削除事故から学ぶ5つの安全対策｜DataTalks.Club事例と防御設定"
emoji: "🛡️"
type: "tech"
topics: ["claudecode", "terraform", "セキュリティ", "devops", "ai"]
published: true
---

Claude Code 本番DB削除事故が世界的に報道されました。DataTalks.Clubの2.5年分のデータがAIコーディングエージェントの`terraform destroy`で消失した事件です。本記事では事故の経緯を整理し、Claude Codeに本番環境を触らせないための具体的な安全対策を5つ紹介します。

## DataTalks.Club事故の経緯

2026年2月26日、データエンジニアリング教育プラットフォーム「DataTalks.Club」の創設者Alexey Grigorev氏が、Claude Codeを使ってインフラ作業中に本番データベースを完全に失う事故が発生しました。

### 何が起きたのか

Grigorev氏は別サービス「AI Shipping Labs」のサイトをAWSに移行し、DataTalks.Clubと同じインフラ上で動かそうとしていました。Claude Code自身は「2つのセットアップを統合するのは避けるべき」と助言していましたが、コスト削減のためにこれを無視して進めました。

致命的だったのは、**PCを切り替えた際にTerraformのステートファイルを移行し忘れた**ことです。ステートファイルがない状態でClaude Codeに重複リソースの削除を指示したところ、エージェントは`terraform destroy`を実行。ステートファイルが記述していたインフラ全体——VPC、RDSデータベース、ECSクラスタ、そして自動スナップショットまで——が消滅しました。

10万人以上の受講者が利用するプラットフォームの宿題提出データ、プロジェクト、リーダーボードを含む**2.5年分・194万行のレコード**が一瞬で失われました。

### 復旧と教訓

AWS Business Supportの支援により、隠れたスナップショットから約24時間で復旧に成功しました。しかしGrigorev氏自身が「AIエージェントにTerraformコマンドを実行させすぎた」と振り返っているように、これは防げた事故でした。

## 安全対策1: permissions.denyで危険コマンドをブロックする

Claude Codeの`settings.json`で特定コマンドの実行を拒否できます。プロジェクトルートの`.claude/settings.json`またはグローバルの`~/.claude/settings.json`に設定します。

```json
{
  "permissions": {
    "deny": [
      "Bash(terraform:destroy*)",
      "Bash(terraform:apply*)",
      "Bash(rm:-rf*)",
      "Bash(aws:rds delete*)",
      "Bash(aws:ec2 terminate*)",
      "Bash(kubectl:delete namespace*)",
      "Bash(docker:system prune*)"
    ]
  }
}
```

### 重要な注意: permissions.denyのバグ

`permissions.deny`には**過去に設定が無視されるバグが報告されています**（GitHub Issue [#4365](https://github.com/anthropics/claude-code/issues/4365)、[#6699](https://github.com/anthropics/claude-code/issues/6699)）。バージョンによっては`deny`ルールが正しく適用されないことがあります。

**必ず最新バージョンのClaude Codeを使い、設定後に意図通りブロックされるかテストしてください。** `permissions.deny`だけに頼らず、次に紹介するhooksとの併用を強く推奨します。

## 安全対策2: hooksによる防御（PreToolUseでの危険コマンドブロック）

Claude Code hooksは`permissions.deny`とは独立した防御レイヤーです。`PreToolUse`イベントで危険なコマンドをブロックするスクリプトを挟み込めます。

`.claude/settings.json`にhooksを設定します。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/your/project/.claude/scripts/block-dangerous-commands.sh"
          }
        ]
      }
    ]
  }
}
```

ブロックスクリプトの例を示します。

```bash
#!/bin/bash
# .claude/scripts/block-dangerous-commands.sh
# PreToolUse hookで呼ばれる。終了コード2で拒否。

INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

# 危険なコマンドパターンを定義
DANGEROUS_PATTERNS=(
  "terraform destroy"
  "terraform apply -auto-approve"
  "rm -rf /"
  "aws rds delete"
  "aws ec2 terminate"
  "DROP DATABASE"
  "DROP TABLE"
  "kubectl delete namespace"
)

for pattern in "${DANGEROUS_PATTERNS[@]}"; do
  if echo "$COMMAND" | grep -qi "$pattern"; then
    echo "BLOCKED: '$pattern' に一致する危険なコマンドです。手動で実行してください。" >&2
    exit 2
  fi
done

exit 0
```

```bash
chmod +x .claude/scripts/block-dangerous-commands.sh
```

**hooksの利点**: `permissions.deny`のバグに影響されず、独自のロジックで柔軟にフィルタリングできます。終了コード`2`を返すとClaude Codeはそのツール呼び出しを拒否し、エラーメッセージをコンテキストに含めます。

## 安全対策3: 本番環境分離（.claude/settings.jsonでの制限）

本番環境に関わるディレクトリやファイルへのアクセス自体を制限するアプローチです。

```json
{
  "permissions": {
    "allow": [
      "Read(*)",
      "Edit(src/**)",
      "Edit(tests/**)",
      "Bash(npm:*)",
      "Bash(yarn:*)",
      "Bash(git:*)"
    ],
    "deny": [
      "Edit(infrastructure/production/**)",
      "Edit(**/.env.production)",
      "Edit(**/terraform.tfstate*)",
      "Bash(terraform:*)",
      "Bash(aws:*)",
      "Bash(gcloud:*)"
    ]
  }
}
```

さらに`.claudeignore`ファイルで、Claude Codeのコンテキストから本番関連ファイルを除外します。

```text
# .claudeignore
# 本番環境のインフラ定義を除外
infrastructure/production/
terraform.tfstate
terraform.tfstate.backup
*.tfvars
.env.production
.env.staging
```

**この分離の考え方**: Claude Codeはアプリケーションコードの開発に使い、インフラ操作は人間が行う。これが最もシンプルで強力な防御です。

## 安全対策4: Terraform/インフラ操作のガードレール

DataTalks.Club事故の直接原因はTerraformの運用にありました。Claude Codeの設定とは別に、Terraform側にもガードレールを設けるべきです。

### ステートファイルのリモート管理

ローカルのステートファイルに依存したことが事故の遠因でした。S3 + DynamoDBによるリモートステート管理は必須です。

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "your-company-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "ap-northeast-1"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}
```

### リソースの削除保護

本番データベースには必ず削除保護を設定します。

```hcl
# rds.tf
resource "aws_db_instance" "production" {
  identifier     = "production-db"
  engine         = "postgresql"
  instance_class = "db.t3.medium"

  # 削除保護: これがあればterraform destroyでも削除されない
  deletion_protection = true

  # スナップショット: 削除時に最終スナップショットを作成
  skip_final_snapshot = false
  final_snapshot_identifier = "production-db-final-snapshot"

  # 自動バックアップの保持期間
  backup_retention_period = 14
}
```

### lifecycle preventでの二重防御

```hcl
resource "aws_db_instance" "production" {
  # ... 上記の設定 ...

  lifecycle {
    prevent_destroy = true
  }
}
```

`prevent_destroy = true`を設定すると、`terraform destroy`や`terraform apply`でそのリソースを削除しようとした時点でTerraformがエラーを出して停止します。

## 安全対策5: バックアップ戦略

「バックアップがあるから大丈夫」は危険です。DataTalks.Clubの事故では、自動スナップショットもインフラと一緒に削除されました。

### 3-2-1バックアップルール

- **3つ**のコピーを保持する
- **2種類**の異なるメディアに保存する
- **1つ**はオフサイト（別リージョン/別アカウント）に置く

### クロスアカウントバックアップの設定例

```bash
# 本番アカウントのRDSスナップショットを別アカウントにコピー
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier arn:aws:rds:ap-northeast-1:111111111111:snapshot:production-daily \
  --target-db-snapshot-identifier production-daily-backup \
  --source-region ap-northeast-1 \
  --kms-key-id arn:aws:kms:ap-northeast-1:222222222222:key/your-key-id
```

### AWS Backupによる自動化

AWS Backupを使えば、RDS・EBS・DynamoDBなどのバックアップを一元管理し、別リージョンや別アカウントへの自動コピーが設定できます。Terraformで削除されても、AWS Backupの保持ポリシーに基づいたバックアップは残ります。

## まとめ: 多層防御が鍵

DataTalks.Club事故は「AIが悪い」のではなく、**AIに与える権限の管理が不十分だった**ことが本質です。

| 対策 | 防御レイヤー | 難易度 |
|------|-------------|--------|
| permissions.deny | Claude Code内部 | 低（バグに注意） |
| hooks（PreToolUse） | Claude Code外部スクリプト | 中 |
| 本番環境分離 | アクセス制御 | 低 |
| Terraformガードレール | インフラ側 | 中 |
| バックアップ戦略 | 最終防衛線 | 中〜高 |

単一の対策に頼るのではなく、**複数のレイヤーを重ねる「多層防御（Defense in Depth）」** の考え方が重要です。Claude Codeは強力な開発ツールですが、本番インフラの操作は人間の手で行う——この原則を守るだけで、大半の事故は防げます。

---

AI開発ツールの安全な使い方や最新のベストプラクティスを毎週お届けしています。実践的なTipsを見逃したくない方は、ぜひニュースレターをご購読ください。

[Byline - AI Dev Tips ニュースレター](https://byline.dev/ai-dev-tips)
