# Lessons Learned

2026-09-09〜2026-09-11 に行った、GPUなし・Windows PC 3台・1GbE・`llama.cpp` RPCによる Llama 3.3 70B Q4_K_M 実証実験から得た教訓をまとめる。

## 1. 70Bは3台のCPU PCでも実際に動く

構成:

```text
R841PC000  i9-13900H / 約32GB RAM / controller + local CPU
R841PC025  i9-13900H / 約32GB RAM / RPC worker
R841PC026  i9-13900H / 約32GB RAM / RPC worker
Network    1GbE
```

短い70B推論は正常完走した。

```text
70B distributed inference success

[ Prompt: 2.1 t/s | Generation: 0.8 t/s ]
```

GPUがなくても、複数台の既存PCを束ねることで70B級モデルを実行できることを確認した。

## 2. 対話用途より夜間バッチ用途に向く

短い応答では `Generation: 0.8 t/s` 程度だったが、長文repo監査では実測が大きく低下した。

最終的な Last Beacon 監査:

```text
Start      : 2026-09-10 17:08:44
Output end : 2026-09-11 03:47:18
Elapsed    : 約10時間38分

Prompt     : 1.1 t/s
Generation : 0.1 t/s
```

この速度ではインタラクティブな利用は厳しい。一方、8時間以上待てる夜間バッチであれば用途は成立しうる。

30台ある場合も、1本の推論を30台へ広げるより、

```text
3 PCs x 10 clusters
```

として独立repoを並列処理する方が有望。

## 3. 長大なrepoを1プロンプトに詰める方式は非効率

Last Beacon の主要21ファイルをまとめた初回入力:

```text
Prompt chars : 71172
Request      : 32802 tokens
```

最初の `-c 32768` では、わずか34 tokens超過して失敗した。

```text
Error: request (32802 tokens) exceeds the available context size (32768 tokens)
```

`-c 49152` へ拡大すると処理自体は完走したが、約10時間38分かかった。

今後は repo 全体を一発投入せず、3k〜6k tokens程度のチャンクに分ける。

```text
repository
   ↓
small chunks
   ↓
focused audits
   ↓
short structured findings
   ↓
aggregation
   ↓
final AUDIT.md
```

## 4. 文字数ではなくtoken数で設計する

日本語、Markdown、TypeScript、JSON等が混在すると、文字数だけでcontext使用量を判断できない。

入力のtoken数だけでなく、

```text
input tokens
+ expected output tokens
+ system/template overhead
< context size
```

となるように余裕を持たせる必要がある。

## 5. prefillが大きなボトルネックになる

短いプロンプトの70Bテストだけを見ると `Prompt: 2.1 t/s` だったが、32k token級の長文では `Prompt: 1.1 t/s` まで低下した。

CPU分散LLMのrepo監査では、「回答を何token生成するか」だけでなく「何万token読ませるか」が総所要時間を大きく左右する。

したがって、各監査passへ必要なコードだけを渡すことが重要。

## 6. 長い自然文を生成させない

最終監査では `Generation: 0.1 t/s` まで低下した。

各チャンクで長いMarkdownを書かせるより、

```text
severity | file | symbol/line | finding | evidence | confidence
```

のような短い構造化結果を返させ、最後の1回だけ文章化する方が効率がよい。

## 7. 70Bでも監査結果をそのまま信用できない

最終監査は完走したが、内容には不適切な重大度判定や根拠の弱い一般論が含まれた。

例:

- Cursor関連の設定が不明、というだけの事項を Critical 扱い
- デプロイ手順が不明、という事項を Critical 扱い
- Zodスキーマについて具体的根拠を示さず「確認が必要」とする
- コード固有の問題ではなく一般的なベストプラクティスへ流れる

したがって監査promptはより強く制約する。

例:

```text
- 実コードの根拠がない指摘は禁止
- 問題が見つからない場合は「問題なし」と書く
- Critical は RCE、認証突破、重大な情報漏えい、データ消失等に限定
- 一般論だけの指摘は禁止
- file / symbol / evidence を必須とする
- confidence を必須とする
```

70Bであっても「自動判定器」ではなく、一次スクリーニングやレビュー補助として扱う。

## 8. ノード電源断を通常の障害として扱う

最初の48k監査は数時間動作した後に以下で終了した。

```text
recv failed
Remote RPC server crashed or returned malformed response
```

後から、学生がRPC worker PCをシャットダウンしていたことが判明した。

これは `llama.cpp` の長時間安定性問題とは区別すべきで、実運用では「workerが途中で落ちる」ことを前提にする必要がある。

必要な仕組み:

- worker死活監視
- チャンク単位チェックポイント
- 完了済みチャンクの再利用
- 失敗チャンクだけ再実行
- 空きクラスタへの再割当
- retry回数と失敗理由の記録

4時間処理した最後にworkerが落ちて全損、という構成は避ける。

## 9. RPC workerは常時同じCPU使用率にはならない

長文監査中にworkerを観測した。

ある時点:

```text
R841PC025
CPU DELTA : 92.4 sec / 10 sec
RAM       : 約13 GiB

R841PC026
CPU DELTA : 0.0 sec / 10 sec
RAM       : 約16.7 GiB
```

数分後:

```text
R841PC026
CPU DELTA : 99.7 sec / 10 sec
RAM       : 約15 GiB
```

一方のworkerが待機している瞬間があっても、それだけで分散失敗とは判断できない。layer splitでは処理フェーズによって担当workerのCPU負荷が移る。

監視は瞬間CPU使用率だけでなく、一定時間のCPU time delta、RAM保持量、RPC疎通を併せて見る。

## 10. LabOpsから長時間処理を直接実行しない

LabOps側PowerShellには600秒タイムアウトがあった。

以下はすべて直接実行では不向きだった。

- llama.cppビルド
- 40GB級GGUFダウンロード
- 70Bロード / 推論
- repo監査

長時間処理はScheduled Taskへ切り離し、LabOpsは以下だけ担当させる。

```text
start job
check status
read result
stop/retry if needed
```

この分離により、クライアントGUIやHTTP要求の寿命とLLMジョブの寿命を切り離せる。

## 11. 終了コードはLLM本体の結果を保存する

初期バッチでは `%ERRORLEVEL%` の保存方法にミスがあり、文字列 `%ERRORLEVEL%` がそのまま残った。

また、Scheduled Taskの `Last Result: 0` だけでは、内部でLLMエラーが出ていることを見逃したケースもあった。

最終的には、

```bat
llama-cli.exe ...
set RC=%ERRORLEVEL%
echo %RC% > exitcode.txt
exit /b %RC%
```

のようにLLM本体の終了コードを明示保存し、stdout / stderr / output artifactも併せて確認する。

## 12. RPCサーバーは再起動後の復旧を考える

学生PCのシャットダウン後、RPC workerは自動的には利用可能状態へ戻らなかった。

再起動後は以下を確認する。

```text
ggml-rpc-server process exists
TCP 50052 LISTEN
controller -> worker Test-NetConnection = True
llama-cli --list-devices recognizes worker
```

長期的には、RPC serverをSYSTEM権限のScheduled Task等で自動起動し、controller側でジョブ開始前にpreflight checkを行う。

## 13. OpenSSLなしビルドでは -hf を使えない

今回のllama.cppビルドではOpenSSL dev filesが無く、HTTPS supportが無効だった。

そのため、

```text
llama-cli -hf ...
```

によるHugging Face直接取得は失敗した。

GGUFは `curl.exe` で取得し、ローカルファイルを `-m` で指定する方式にした。

これはモデル管理の再現性という意味でも扱いやすかった。

## 14. 1GbEでも動くが、ロードと長文処理は重い

1GbEでも70B分散推論そのものは成立した。

一方で、40GB級のモデルをRPC workerへ配布しながら利用するため、モデルロードや長文処理には時間がかかる。

今後の改善候補:

- RPC cacheの利用
- モデル常駐
- 2.5GbE / 10GbEとの比較
- tensor split / layer配置の比較
- 同一モデルを複数ジョブで再利用するserver構成

## 15. 最終的な監査パイプライン案

現時点で有望なのは次の形。

```text
central scheduler
      ↓
git clone / pull
      ↓
repository inventory
      ↓
token-aware chunking
      ↓
focused audit jobs
      ↓
checkpoint each finding
      ↓
retry failed chunks
      ↓
aggregate findings
      ↓
final 70B synthesis
      ↓
AUDIT-YYYY-MM-DD.md
      ↓
dedicated branch
      ↓
commit / push / optional PR
```

LLMにはGit権限や自由なファイル操作を持たせず、監査結果生成だけを担当させる。Git操作は決め打ちスクリプト側で行う。

## 結論

今回のPoCで確認できたこと:

- GPUなしのi9-13900H級PC 3台でLlama 3.3 70B Q4_K_Mは動く
- 短い推論は `Generation: 0.8 t/s` を確認
- 約32.8k tokenの実repo入力を48k contextで最後まで処理できた
- 実repo監査を約10時間38分で完走した
- ただし長文一括監査は遅く、出力品質もそのままでは不十分
- worker電源断で長時間ジョブが全損するため、チェックポイントとretryが必須
- 実運用では3台1クラスタを多数並列化する夜間バッチ方式が有望

次の段階は「70Bを動かせるか」ではなく、**壊れにくく、根拠のある監査結果を、限られたtokenと時間でどう作るか**というパイプライン設計になる。
