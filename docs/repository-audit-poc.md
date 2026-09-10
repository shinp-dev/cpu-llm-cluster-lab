# リポジトリ監査PoC

## 目的

ローカルに置いたリポジトリを、3台CPUで動作するLlama 3.3 70B Q4_K_Mへ渡し、コード監査レポートをMarkdownとして出力する。

今回の対象:

```text
C:\dev\Last_Beacon-main
```

出力予定:

```text
C:\dev\Last_Beacon-main\AI-AUDIT.md
```

## 監査方針

対象ファイルを無制限に丸ごと投入するのではなく、以下を除外する。

- `.git`
- `node_modules`
- `dist`
- `build`
- `bin`
- `obj`
- `coverage`
- `.next`
- `vendor`
- `target`
- `.env*`
- secret / credential / private-keyを名前に含むファイル

主要ソース、設定ファイル、README等を集め、監査用promptへまとめる。

## 初回PoCの入力

```text
Files found : 21
Prompt chars: 71172
```

当初は入力長を文字数で制限していたが、実際のllama.cppでは次のエラーになった。

```text
Error: request (32802 tokens) exceeds the available context size (32768 tokens)
```

わずか34 tokensの超過だった。

## 重要な知見

### 1. 文字数ではなくtoken数で管理する

コード、Markdown、日本語が混在するrepoでは文字数からtoken数を正確に予測できない。

監査基盤では、入力を組み立てた後にtoken数を見積もるか、十分なcontext余白を設ける必要がある。

### 2. 出力領域を先に確保する

入力がcontext上限ぎりぎりだと、監査レポートを生成する余地が無い。

今回の再試行では:

```text
input     : 約32802 tokens
context   : 49152 tokens
max output: 2500 tokens
```

とした。

### 3. 一括監査はPoC向け

本格運用ではrepo全体を1回で投入するより、以下の多段構成が適している。

```text
1. repository structure scan
2. architecture / entrypoint review
3. directory or file chunk reviews
4. security-focused pass
5. bug/error-handling pass
6. test-gap pass
7. findings aggregation
8. final AUDIT.md
```

### 4. LLMにはGit操作を任せなくてもよい

LLMは監査結果だけを返し、clone / pull / file collection / commit / pushは決め打ちスクリプトで行う方が安全で再現しやすい。

想定フロー:

```text
git clone / pull
      ↓
source collection
      ↓
70B audit passes
      ↓
AUDIT.md
      ↓
git add / commit / push
```

自動pushを行う場合も、最初はmain直書きではなく専用branchへ監査Markdownのみを追加する運用が安全。

## 監査観点

PoCでは以下を重点項目として指定した。

1. 致命的または重大なバグ
2. セキュリティ上の問題
3. 認証・認可・権限制御
4. 入力検証
5. 個人情報・秘密情報・トークン等の扱い
6. DBアクセスおよびデータ整合性
7. API設計とエラー処理
8. 非同期処理・競合・状態管理
9. フロントエンドとバックエンドの責務
10. 例外処理・障害時の挙動
11. テスト不足
12. 保守性・重複・複雑性
13. デプロイ時に問題になりそうな点
14. 実装とREADME/設計意図の食い違い

また、根拠がコードから確認できない場合は「要確認」とし、推測だけで重大問題と断定しないよう指示した。

## 最終的に目指す形

昼間は授業用PCとして利用し、夜間は複数の70Bクラスタとしてコード監査バッチへ転用する。

```text
30 PCs
  ↓
10 x (3-PC 70B cluster)
  ↓
central audit job queue
  ↓
multiple repositories in parallel
  ↓
AUDIT-YYYY-MM-DD.md
  ↓
commit / push
```

単一推論の速度向上よりも、独立したrepo監査を多数並列処理することで計算資源を活かす構想。
