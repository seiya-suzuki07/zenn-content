---
title: "【Cursor】コードをぶっ壊された時のリカバリ・復元方法と破壊を防ぐワークフロー"
emoji: "🛡️"
type: "tech"
topics: ["Cursor", "Git", "AIエディタ", "リカバリ", "開発効率化"]
published: true
---

CursorのAI編集でコードをぶっ壊された経験はありませんか？ Gitの差分確認なしにAcceptしてしまい、元に戻せなくなるケースが頻発しています。本記事では、Cursorでコードが壊された時の具体的なリカバリ・復元手順と、事前にコード破壊を防ぐワークフローを解説します。

## なぜCursorでコードが壊されるのか

Cursorはコードベース全体を理解した上で編集を提案しますが、以下のようなケースでコードが意図せず破壊されます。

- **Agent modeによる大規模な自動編集**: 複数ファイルを一括変更し、依存関係を壊す
- **差分を確認せずにAccept**: AI提案を鵜呑みにして、削除されたロジックに気づかない
- **Format On Saveとの競合**: Cursorの編集とフォーマッターが衝突し、コードがサイレントに上書きされる
- **Agent Review Tabとの競合**: ファイルロッキングの競合で変更がサイレントにリバートされる

2026年3月時点で、Cursor公式が確認している3つの根本原因は「Agent Review conflict」「Cloud Sync conflict」「Format On Save conflict」です。

## Cursorでコードを壊された時のリカバリ手順

### 方法1: Cursorのチェックポイント機能で復元する

Cursorには各AI対話ステップでコードベースのスナップショットを自動保存する**チェックポイント機能**が備わっています。

1. 左側のチャットパネルを開く
2. 問題が発生する前のメッセージを探す
3. メッセージ右側の「**Restore checkpoint**」ボタンをクリック
4. 「Undo all changes up to this checkpoint」を選択

これが最も手軽な復元方法です。ただし、チャット履歴を閉じたり新しいセッションを開始すると、チェックポイントが失われる可能性があります。

### 方法2: Gitで直前の状態に戻す

チェックポイントが使えない場合、Gitが最後の砦になります。

**直前のコミットまで全て戻す場合:**

```bash
# 変更を全て破棄して直前のコミットに戻す
git checkout -- .

# ステージングも含めて完全にリセット
git reset --hard HEAD
```

**特定のファイルだけ戻す場合:**

```bash
# 特定ファイルを直前のコミットの状態に復元
git checkout HEAD -- src/components/App.tsx

# 複数ファイルを復元
git checkout HEAD -- src/lib/auth.ts src/utils/helpers.ts
```

**数コミット前の状態に戻す場合:**

```bash
# コミット履歴を確認
git log --oneline -10

# 特定のコミットの状態に戻す（履歴を残す形で）
git revert HEAD~3..HEAD
```

### 方法3: git stashで退避してから調査する

「壊されたかもしれないが確信がない」場合に有効です。

```bash
# 現在の変更を一時退避
git stash

# アプリが正常に動くか確認
npm run dev  # or yarn dev

# 問題なければ退避した変更を戻す
git stash pop

# 壊れていたなら退避した変更を破棄
git stash drop
```

### 方法4: git reflogで「消えた」コミットを復元する

`git reset --hard` を実行してしまった後でも、reflogから復元できます。

```bash
# reflogで過去の状態を確認
git reflog

# 例: 3つ前の状態に戻る
git checkout HEAD@{3}

# そこから新しいブランチを作成
git checkout -b recovery-branch
```

### 方法5: duraで継続的なスナップショットを取る

通常のGitコミットでは頻度が足りない場合、**dura**というツールが有効です。ファイル変更のたびにバックグラウンドでGitコミットを自動生成します。

```bash
# duraのインストール（Rust製）
cargo install dura

# バックグラウンドで監視開始
dura serve &

# 監視対象のリポジトリを登録
cd /your/project && dura watch
```

duraが作成したコミットは通常のブランチには影響せず、専用のrefで管理されるため、安心して使えます。

## コード破壊を防ぐワークフロー

リカバリ方法を知っておくことも大切ですが、そもそも壊されないワークフローを構築することが最優先です。

### 1. AI編集用の作業ブランチを必ず切る

```bash
# AI作業前に必ずブランチを作成
git checkout -b ai/feature-name

# 作業が完了したらdiffを確認してからマージ
git diff main...ai/feature-name
```

mainブランチで直接AI編集を行うのは最も危険なパターンです。ブランチを切っておけば、最悪の場合でもブランチごと削除するだけで済みます。

### 2. AI編集の前にこまめにコミットする

```bash
# AI編集を依頼する前に、現在の作業をコミット
git add -A && git commit -m "checkpoint: before AI refactor"
```

このチェックポイントコミットがあれば、`git reset --hard HEAD~1` で即座にAI編集前の状態に戻れます。

### 3. Acceptする前にdiffを必ず確認する

Cursorで提案された変更をAcceptする前に：

- **削除された行（赤い行）を重点的に確認する** — AIは「不要」と判断したコードを無断で削除することがある
- **import文の変更を確認する** — 使用中のimportが削除されていないか
- **関数のシグネチャ変更を確認する** — 引数や戻り値の型が変わっていないか

### 4. Format On Saveを一時的に無効化する

Cursorの既知のバグ対策として、AI編集中はFormat On Saveを無効化します。

```json
// .vscode/settings.json
{
  "editor.formatOnSave": false
}
```

### 5. .cursorrulesで破壊的変更を抑制する

プロジェクトルートに `.cursorrules` ファイルを置き、AIの振る舞いを制御します。

```text
# .cursorrules
- 既存のコードを削除する前に、必ず理由を説明してください
- 関数のシグネチャを変更する場合は、呼び出し元も全て更新してください
- importを削除する場合は、そのimportが他で使われていないことを確認してください
- 一度に変更するファイルは3つ以下にしてください
```

### 6. CI/テストを活用して壊れたことに早く気づく

```bash
# AI編集後に即テスト実行
npm test

# 型チェック
npx tsc --noEmit

# リント
npx eslint src/
```

これらをpre-commitフックに設定しておけば、壊れたコードがコミットされること自体を防げます。

## まとめ: Cursorとの安全な付き合い方

| 状況 | 対処法 |
|---|---|
| Accept直後に気づいた | Cmd+Z / Ctrl+Z で即Undo |
| チャット中に気づいた | チェックポイントで復元 |
| コミット前に気づいた | `git checkout -- .` で全戻し |
| コミット後に気づいた | `git revert` で履歴を残して戻す |
| resetしてしまった | `git reflog` で救出 |
| そもそも防ぎたい | ブランチ戦略 + こまめなコミット + diff確認 |

CursorのようなAIエディタは生産性を大幅に向上させますが、「AIの提案を無条件に信頼しない」という姿勢が不可欠です。ブランチを切る・コミットする・diffを確認する。この3つの習慣だけで、コード破壊のリスクは劇的に下がります。

---

:::message
**AI開発の最新Tipsを毎週配信中** -- Cursorの安全な使い方、Git戦略、AIエージェント開発のベストプラクティスなど、現場で使える情報をニュースレター「Byline」でお届けしています。
👉 [Bylineに登録する](https://byline.dev/ai-dev-tips)
:::
