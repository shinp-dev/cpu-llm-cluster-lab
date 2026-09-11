# リポジトリ監査PoC

## 目的

ローカルに置いたリポジトリを、3台CPUで動作するLlama 3.3 70B Q4_K_Mへ渡し、コード監査レポートをMarkdownとして出力する。

今回の対象:

```text
C:\dev\Last_Beacon-main
```

出力:

```text
C:\dev\Last_Beacon-main\AI-AUDIT.md
```

## 初回PoCの入力

`node_modules`, `.git`, build成果物、`.env`等を除外し、主要コード・設定・READMEを収集。

```text
Files found  : 21
Prompt chars : 71172
Request      : 32802 tokens
```

## 32k contextでは失敗

当初:

```text
-c 32768
-n 2500
```

で実行したが、以下で失敗。

```text
Error: request (32802 tokens) exceeds the available context size (32768 tokens)
```

わずか34 tokensの超過だった。

## 48k contextへ拡張

再試行条件:

```text
-c 49152
-n 2500
```

これにより入力と出力の余白を確保した。

## 1回目の48k監査: worker電源断

数時間処理した後、

```text
recv failed
Remote RPC server crashed or returned malformed response
```

で停止。

後から、学生がRPC worker PCをシャットダウンしていたことが判明した。

長時間ジョブでは、worker障害を前提としたcheckpoint / retryが必須である。

## 2回目の48k監査: 完全完走

再実行開始:

```text
2026-09-10 17:08:44
```

出力完了:

```text
2026-09-11 03:47:18
```

総所要時間:

```text
約10時間38分
```

終了状態:

```text
Status      : Ready
Last Result : 0
Exit Code   : 0
STDERR      : empty
```

出力:

```text
AI-AUDIT.md
6146 bytes
```

実測:

```text
[ Prompt: 1.1 t/s | Generation: 0.1 t/s ]
```

## 監査品質の評価

技術的には完走したが、結果品質はそのまま自動採用できる水準ではなかった。

問題例:

- 根拠の弱い事項をCriticalに分類
- 「設定が不明」「確認が必要」といった曖昧な内容を重大問題として扱う
- repo固有の問題より一般的なベストプラクティスへ流れる
- severity calibrationが不十分

したがって、次版では「自由な監査レポート」を直接生成させず、根拠付きの短いfindingを積み上げる。

## 次版の監査方式

```text
repository inventory
      ↓
token-aware chunking
      ↓
3k〜6k token focused audit
      ↓
structured finding
      ↓
checkpoint
      ↓
retry on failure
      ↓
aggregation
      ↓
final report generation
```

各findingは少なくとも以下を持つ。

```text
severity
file
symbol / line
finding
evidence
impact
confidence
```

## Promptの改善方針

次回は以下を明示する。

```text
- 実コードの根拠がない指摘は禁止
- 問題がなければ「問題なし」
- file / symbol / evidence を必須
- CriticalはRCE、認証突破、重大な情報漏えい、データ消失級に限定
- 一般的なベストプラクティスだけの指摘は禁止
- confidenceを必須
```

## Git操作との分離

LLMはコード監査とfindings生成のみを担当する。

```text
git clone / pull
      ↓
source collection
      ↓
LLM audit
      ↓
AUDIT.md
      ↓
git add / commit / push
```

clone / pull / commit / pushは決め打ちスクリプトで行い、LLMへ自由なGit操作権限を与えない。

## 障害復旧

実運用では以下を入れる。

- workerのpreflight check
- RPC死活監視
- chunk完了時の永続化
- 失敗chunkのみretry
- 空きclusterへの再割当
- stdout / stderr / exit code保存
- worker再起動後のRPC自動復旧

## 最終的に目指す形

```text
30 PCs
  ↓
10 x (3-PC 70B cluster)
  ↓
central audit job queue
  ↓
multiple repositories in parallel
  ↓
chunked / fault-tolerant audits
  ↓
AUDIT-YYYY-MM-DD.md
  ↓
commit / push
```

単一推論を高速化するより、夜間に複数repoを独立並列処理する方向を重視する。

より広い教訓は [Lessons Learned](lessons-learned.md) にまとめる。
