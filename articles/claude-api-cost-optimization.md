---
title: "Claude APIのトークン節約術 - プロンプトキャッシュとバッチAPIで最大95%コスト削減"
emoji: "💰"
type: "tech"
topics: ["Claude", "API", "コスト削減", "Anthropic", "LLM"]
published: true
---

Claude APIの従量課金コストが想定以上にかかっていませんか？プロンプトキャッシュ（キャッシュ読込90%オフ）やバッチAPI（50%割引）を活用すれば、Claude APIのトークン消費を大幅に節約し、最大95%のコスト削減が可能です。本記事では、2026年3月時点の最新料金体系をもとに、具体的な実装方法とモデル選択の判断基準を解説します。

## Claude APIの最新料金体系（2026年3月時点）

まずは現行モデルの料金を把握しましょう。

| モデル | 入力（/1Mトークン） | 出力（/1Mトークン） | 特徴 |
|--------|---------------------|---------------------|------|
| **Opus 4.6** | $5.00 | $25.00 | 最高性能。複雑な推論・コード生成向き |
| **Sonnet 4.6** | $3.00 | $15.00 | 性能とコストのバランスが最良 |
| **Haiku 4.5** | $1.00 | $5.00 | 高スループット・低コスト向き |

> Sonnet 4.6は200Kトークン超の入力時、$6.00/$22.50に料金が上がる点に注意してください。

### モデル選択の判断基準

- **Opus 4.6**: 法律文書の分析、高度なコード生成、マルチステップの推論タスクなど「品質を妥協できない」場面で使う
- **Sonnet 4.6**: 一般的なチャットボット、要約、翻訳、コードレビューなど大多数のユースケースに最適。Opusの60%の料金で十分高品質な出力が得られる
- **Haiku 4.5**: 分類、感情分析、簡単な抽出タスクなど、大量リクエストを低コストでさばきたいケース

## プロンプトキャッシュで入力コスト90%削減

プロンプトキャッシュは、繰り返し送信するシステムプロンプトや参照文書をサーバー側にキャッシュし、2回目以降の読み込みコストを**基本料金の10%**に抑える機能です。

### キャッシュの料金構造

| 操作 | コスト倍率 | 説明 |
|------|-----------|------|
| キャッシュ書き込み（5分TTL） | 1.25x | 初回リクエスト時の書き込みコスト |
| キャッシュ書き込み（1時間TTL） | 2.0x | 長寿命キャッシュの書き込みコスト |
| キャッシュ読み込み | 0.1x | 2回目以降のリクエスト |
| 通常入力 | 1.0x | キャッシュ対象外の部分 |

つまり、同じシステムプロンプトで10回リクエストを送る場合、キャッシュなしなら10回分のフル料金がかかりますが、キャッシュありなら「1.25回分 + 0.1回分 x 9 = 2.15回分」で済みます。**約78%の削減**です。

### 最小キャッシュトークン数

- Sonnet 4.6 / Sonnet 4.5: **1,024トークン**以上
- Opus 4.6 / Haiku 4.5: **4,096トークン**以上

### Pythonでの実装例

```python
import anthropic

client = anthropic.Anthropic()

# 長いシステムプロンプト（例: 製品マニュアル全文）
system_prompt = "あなたは製品サポートAIです。以下のマニュアルを参照して回答してください。\n\n" + long_manual_text

# 自動キャッシュ: cache_controlをトップレベルに指定するだけ
response = client.messages.create(
    model="claude-sonnet-4-5-20241022",
    max_tokens=1024,
    cache_control={"type": "ephemeral"},  # 5分間キャッシュ
    system=system_prompt,
    messages=[
        {"role": "user", "content": "返品ポリシーを教えてください"}
    ]
)

# キャッシュ使用状況の確認
usage = response.usage
print(f"キャッシュ書き込み: {usage.cache_creation_input_tokens} tokens")
print(f"キャッシュ読み込み: {usage.cache_read_input_tokens} tokens")
print(f"通常入力: {usage.input_tokens} tokens")
```

明示的にキャッシュ対象を指定する方法もあります。

```python
# 明示的キャッシュ: 特定のブロックにcache_controlを付与
response = client.messages.create(
    model="claude-sonnet-4-5-20241022",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": system_prompt,
            "cache_control": {"type": "ephemeral"}  # このブロックをキャッシュ
        }
    ],
    messages=[
        {"role": "user", "content": "保証期間は何年ですか？"}
    ]
)
```

## バッチAPIで50%コスト削減

リアルタイム応答が不要な処理（データ分析、コンテンツ生成、大量分類など）には、**バッチAPI**が最適です。通常料金の50%割引で処理できます。

### バッチAPI料金比較

| モデル | 通常入力 | バッチ入力 | 通常出力 | バッチ出力 |
|--------|---------|-----------|---------|-----------|
| Opus 4.6 | $5.00 | $2.50 | $25.00 | $12.50 |
| Sonnet 4.6 | $3.00 | $1.50 | $15.00 | $7.50 |
| Haiku 4.5 | $1.00 | $0.50 | $5.00 | $2.50 |

### Pythonでの実装例

```python
import anthropic
import time

client = anthropic.Anthropic()

# バッチリクエストの作成（最大10,000件）
batch = client.messages.batches.create(
    requests=[
        {
            "custom_id": "review-001",
            "params": {
                "model": "claude-sonnet-4-5-20241022",
                "max_tokens": 512,
                "messages": [
                    {"role": "user", "content": "以下のレビューの感情を分析してください: 素晴らしい製品です！"}
                ]
            }
        },
        {
            "custom_id": "review-002",
            "params": {
                "model": "claude-sonnet-4-5-20241022",
                "max_tokens": 512,
                "messages": [
                    {"role": "user", "content": "以下のレビューの感情を分析してください: 期待外れでした。"}
                ]
            }
        },
        # ... 必要な数だけリクエストを追加
    ]
)

print(f"バッチID: {batch.id}")
print(f"ステータス: {batch.processing_status}")

# 処理完了を待つ（通常1時間以内、最大24時間）
while True:
    status = client.messages.batches.retrieve(batch.id)
    if status.processing_status == "ended":
        break
    time.sleep(60)

# 結果の取得
for result in client.messages.batches.results(batch.id):
    print(f"ID: {result.custom_id}")
    if result.result.type == "succeeded":
        print(f"応答: {result.result.message.content[0].text}")
    else:
        print(f"エラー: {result.result.error}")
```

## キャッシュ + バッチ併用で最大95%削減

プロンプトキャッシュとバッチAPIは**併用可能**です。組み合わせた場合の入力コストを計算してみましょう。

### コスト試算（Sonnet 4.6、入力100万トークン x 100リクエスト）

| 方式 | 計算式 | 合計コスト |
|------|--------|-----------|
| 通常API | $3.00 x 100 | **$300.00** |
| キャッシュのみ | $3.75 + ($0.30 x 99) | **$33.45** |
| バッチのみ | $1.50 x 100 | **$150.00** |
| キャッシュ + バッチ | $1.875 + ($0.15 x 99) | **$16.73** |

通常APIと比較して、キャッシュ+バッチ併用なら**約94%の削減**を実現できます。

> 上記はシステムプロンプト部分（100万トークン）がすべてキャッシュ対象で、ユーザーメッセージ部分のコストを除いた入力コストの比較です。

## コスト削減チェックリスト

実際のプロジェクトで活用するためのチェックリストです。

1. **システムプロンプトが1,024トークン以上** → プロンプトキャッシュを導入
2. **リアルタイム応答が不要なタスク** → バッチAPIに切り替え
3. **同じプロンプトで繰り返しリクエスト** → キャッシュ + バッチ併用
4. **タスクの複雑さに応じたモデル選択** → 分類はHaiku、一般業務はSonnet、高度な推論はOpus
5. **出力トークン数の制御** → `max_tokens`を必要最小限に設定
6. **レスポンスのusageフィールド** → キャッシュヒット率を定期的にモニタリング

## まとめ

Claude APIのコスト最適化は、正しい手法を知っていれば難しくありません。

- **プロンプトキャッシュ**: 繰り返しの入力を90%オフで読み込み
- **バッチAPI**: 非リアルタイム処理を50%割引で実行
- **両者の併用**: 最大95%のコスト削減を実現
- **モデル選択の最適化**: タスクに応じてOpus/Sonnet/Haikuを使い分け

まずは最もコストがかかっているAPIコールを特定し、キャッシュ導入から始めてみてください。

---

**AI開発の最新Tipsを毎週配信中。** Bylineニュースレターでは、Claude APIの実践的な活用法やコスト最適化テクニック、最新のAI開発トレンドをお届けしています。[今すぐ購読する](https://byline.dev/ai-dev-tips)

---

**参考資料:**
- [Claude API Pricing - 公式ドキュメント](https://platform.claude.com/docs/en/about-claude/pricing)
- [Prompt Caching - 公式ドキュメント](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Batch Processing - 公式ドキュメント](https://platform.claude.com/docs/en/build-with-claude/batch-processing)
- [Prompt Caching Cookbook](https://github.com/anthropics/anthropic-cookbook/blob/main/misc/prompt_caching.ipynb)
