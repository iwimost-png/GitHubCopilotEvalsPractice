# GitHub Copilot Evals 実践ガイド & サンプルプロジェクト

2026年11月17日にパブリックプレビューとなった **GitHub Copilot Evals** は、GitHub Copilot のカスタマイズ（**エージェントスキル**、**カスタムエージェント**、**MCPツール**）の動作を、同一条件で繰り返し検証・自動テストするための公式評価フレームワークです。

本リポジトリは、GitHub Copilot Evals の実践的な使い方を体験・理解できるように構成されたサンプルプロジェクトです。

---

## 1. GitHub Copilot Evals とは？

AIエージェントやスキルの振る舞いは、モデルの更新やプロンプト（指示文）の微調整、ツールの追加によって変化します。
**GitHub Copilot Evals** を導入することで、以下のことが可能になります：

- **再現性のあるテスト（Eval）**: リアルなタスクをエージェントに与え、期待通りのツールを呼んでいるか、意図した出力・ファイル変更を行っているかを検証。
- **リグレッション（品質低下）の検知**: プロンプトやスキルの改修前後に同じテストを実行し、意図しない挙動変化を即座に発見。
- **評価エンジン**: Microsoft が主導するエージェント評価基盤 **Vally (`@microsoft/vally-cli`)** がエンジンとして採用されています。

---

## 2. 本リポジトリの構成

本サンプルでは、**Conventional Commits 規約に従ったGitコミットメッセージを生成するスキル** を対象とした評価環境を構築しています。

```
.
├── .github/
│   └── workflows/
│       └── evals-ci.yml                 # GitHub Actions による CI 自動テスト
├── skills/
│   └── conventional-commit/
│       └── SKILL.md                     # 【テスト対象】Copilotカスタムスキル
├── evals/
│   ├── .vally.yaml                      # Vally / Copilot Evals スイート設定
│   ├── commit-helper.eval.yaml          # 【評価定義】プロンプトと合格判定ルール
│   └── fixtures/
│       └── sample.diff                  # テスト用入力データ (git diff)
├── package.json                         # ローカル実行用スクリプト
└── README.md
```

---

## 3. GitHub Copilot Evals の使い方（3つのアプローチ）

GitHub Copilot Evals は、開発スタイルに合わせて3つの方法で利用できます。

### ① VS Code 拡張機能（GUI / Evals Canvas）

1. **拡張機能のインストール**:
   - VS Code の拡張機能マーケットプレイスで `GitHub.copilot-evals` を検索してインストールします。
2. **Evals Canvas の活用**:
   - サイドバーに「Evals」パネルが追加されます。
   - プロジェクト内のスキルやMCPを選択し、GUI上でテストケース（Stimuli）の自動作成、実行、結果のビフォーアフター比較が直感的に行えます。

---

### ② GitHub Copilot CLI / App プラグイン（対話式 / `/eval` コマンド）

Copilot CLI または GitHub Copilot App では、専用のプラグインをインストールして `/eval` スラッシュコマンドを利用できます。

#### インストールコマンド
```bash
# プラグインマーケットプレイスを追加
copilot plugin marketplace add github/evals

# Copilot Evals プラグインをインストール
copilot plugin install eval@github-copilot-evals
```

#### 主なスラッシュコマンド
| コマンド | 説明 | 例 |
|---|---|---|
| `/eval setup` | プロジェクト内の評価ディレクトリや設定ファイルを初期化 | `/eval setup` |
| `/eval create` | スキルやエージェントに対する新しいEvalを作成 | `Create an eval for this skill that checks output format` |
| `/eval run` | 作成したEvalを実行（ローカルでエージェントを動かして合否判定） | `/eval run` |
| `/eval review` | 実行結果や失敗原因・トレースログを確認 | `/eval review` |
| `/eval improve` | テストケースの追加や仕様の改善 | `/eval improve` |
| `/eval compare` | スキル改修前後の2つの実行結果を比較 | `/eval compare` |

---

### ③ GitHub Actions CI/CD（自動テスト & PRコメント）

Pull Request が作成された際に、自動的に Lint と Eval を実行し、PR にテスト結果をコメントするワークフローです。

#### ワークフロー定義例 (`.github/workflows/evals-ci.yml`)
```yaml
name: Copilot Evals CI

on:
  pull_request:
    paths:
      - 'evals/**'
      - 'skills/**'
      - '.github/workflows/evals-ci.yml'
  workflow_dispatch:

permissions:
  contents: read

jobs:
  evals:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      copilot-requests: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - name: Run GitHub Copilot Evals
        uses: github/evals-action@v0
        with:
          working-directory: evals
          suite: smoke
```

> **ポイント**: `github/evals-action@v0` が自動で Vally エンジンをセットアップし、テストを実行後、PR に合否サマリを sticky コメントとして投稿します。

---

## 4. Eval 仕様ファイル（`eval.yaml`）の書き方

`evals/commit-helper.eval.yaml` の中身を例に解説します：

```yaml
name: conventional-commit-eval
description: Evaluate that the conventional-commit skill generates standard commit messages.
type: capability

defaults:
  executor: copilot-sdk
  runs: 1
  timeout: 60s

# テスト実行環境にロードするスキルや入力ファイル
agent_environment:
  skills:
    - ../skills/conventional-commit
  files:
    - src: ./fixtures/sample.diff
      dest: changes.diff

scoring:
  threshold: 1.0  # 1.0（100%合格）を基準値に設定

# テストケース（刺激：Stimuli）
stimuli:
  - name: generate-fix-commit
    prompt: |
      Generate a Git commit message using the conventional-commit skill for the changes in changes.diff.
      Follow Conventional Commits strictly. Output only the commit message.

    # 合否判定器（Graders）
    graders:
      # 1. 指定のスキルが実際に呼び出されたかを検証
      - type: skill-invocation
        name: conventional-commit-invoked
        config:
          required:
            - conventional-commit

      # 2. 出力が正規表現（fix: ... 形式）に合致しているかを検証
      - type: output-matches
        name: matches-conventional-format
        config:
          pattern: '^fix(\([a-zA-Z0-9_-]+\))?:\s+[a-z0-9].+'
```

### 利用できる主な Grader（判定器）
- **`skill-invocation`**: 指定したスキルが適切にエージェントによって選択・実行されたかを検証。
- **`output-matches`**: モデルの回答文字列が正規表現や特定の文字列を満たしているか検証。
- **`file-exists` / `file-matches`**: エージェントが作業ディレクトリ内に正しくファイルを生成・編集したかを検証。
- **`run-command`**: コマンド（テストランナーや構文チェッカー）を実行して終了コード0で判定。
- **`prompt` (LLM-as-judge)**: 自然言語の評価基準（`rubric`）に基づき、別のモデルが定性的な品質（トーンや分かりやすさなど）を採点。

---

## 5. ローカルでの動作確認（クイックスタート）

本リポジトリでは、`@microsoft/vally-cli` を使ってローカルで静的検証（Lint）をすぐに試せます。

```bash
# 依存パッケージのインストール
npm install

# スキル定義（SKILL.md）の静的チェック
npm run lint:skills

# 評価仕様（eval.yaml）の静的チェック
npm run lint:evals

# まとめてチェック
npm run lint
```

実行結果例：
```text
$ npm run lint

> vally lint skills
✅ conventional-commit (2/2 checks passed)
1 skill(s) linted, 1 passed

> vally lint -e evals/commit-helper.eval.yaml
✔ commit-helper.eval.yaml is valid
```
