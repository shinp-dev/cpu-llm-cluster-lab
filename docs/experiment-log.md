# 実験ログ

## 2026-09-09: 3台CPUで70Bを動かす

### 目的

R841PC000を親機、R841PC025 / R841PC026を子機として、GPUなし・CPUのみで70Bモデルを分散実行できるか確認する。

### 初期構成

```text
R841PC000  10.40.112.67  parent/controller
R841PC025  10.40.112.55  RPC worker
R841PC026  10.40.112.58  RPC worker
```

各PCはCore i9-13900H、約32GB RAM。ネットワークは1GbE。

### 1. CMake不足

最初の `git clone` は成功したが、親機で `cmake` がPATH上に存在しなかった。`winget` も利用できなかったため、CMake 4.4.3 portableを配置した。

```text
C:\llm\tools\cmake-4.4.3-windows-x86_64\bin\cmake.exe
```

Visual Studio 2022 Build Toolsは既に存在していた。

```text
C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools
```

### 2. llama.cpp configure

`GGML_RPC=ON` でconfigure。

```text
-- Using RPC backend
-- Including RPC backend
```

CPU backendはAVX2/FMA/F16C。AVX512検出失敗は13900Hでは問題なし。

### 3. フルビルドが600秒でタイムアウト

LabOps側のPowerShell実行が600秒でタイムアウトした。ただしコンパイル自体は進行しており、`ggml-rpc-server.exe` は既に生成済みだった。

必要ターゲットだけに絞って再ビルド。

```powershell
& $cmakeExe --build build --config Release --target llama-cli ggml-rpc-server --parallel 8
```

成功。

### 4. 子機を同一commitに固定

3台すべて以下へ固定。

```text
1945e092030f8668ff93382799502d01490e564d
```

R841PC025 / R841PC026では `ggml-rpc-server` のみビルドした。

### 5. RPC server起動

子機はTCP 50052でRPC serverを起動。

RPCを `0.0.0.0` にbindするとllama.cppから以下の警告が出る。

```text
WARNING: Host ('0.0.0.0') is != '127.0.0.1'
Never expose the RPC server to an open network!
This is an experimental feature and is not secure!
```

そのためWindows Firewallでは親機 10.40.112.67 からのみ50052/tcpを許可。

LabOpsでは長時間プロセスを起動するとGUI側が「実行中」のままになることがあり、ログインの有無ではなく子プロセスの寿命・標準入出力の扱いが原因だった。

### 6. 親機から疎通確認

```text
R841PC025 : True
R841PC026 : True
```

`--list-devices`:

```text
Available devices:
 RPC0: 10.40.112.55:50052 (32398 MiB, 23493 MiB free)
 RPC1: 10.40.112.58:50052 (32398 MiB, 26743 MiB free)
```

### 7. Hugging Face直接取得に失敗

最初は `llama-cli -hf` でGemmaを取得しようとしたが、ビルド時にOpenSSLが無かったため失敗。

```text
error: --model is required
get_repo_commit: error: HTTPS is not supported
```

方針変更し、`curl.exe` でGGUFを取得して `-m` で読み込むようにした。

### 8. Gemma 3 1Bで実推論成功

約768.7MiBの `gemma-3-1b-it-Q4_K_M.gguf` を取得。

結果:

```text
分散推論テスト成功
[ Prompt: 49.0 t/s | Generation: 16.8 t/s ]
```

### 9. 70Bモデル取得

`Llama-3.3-70B-Instruct-Q4_K_M.gguf` を親機へ取得。

LabOpsの600秒制限で直接curlは途中停止したが、その時点で約20.66GiBまで取得済み。Scheduled Taskにcurlを切り離し、`-C -` で再開した。

途中:

```text
23.37 GiB -> 23.61 GiB
10秒で +246.45 MiB
```

最終:

```text
Size : 39.60 GiB
Status: Ready
Last Result: 0
```

### 10. 70B初回実行

LabOpsから直接実行すると600秒を超えたため、推論もScheduled Taskへ切り離した。

構成:

```text
llama-cli on R841PC000
  + local CPU
  + RPC0 R841PC025
  + RPC1 R841PC026
```

70Bテスト結果:

```text
70B distributed inference success
[ Prompt: 2.1 t/s | Generation: 0.8 t/s ]
```

GPUなし、1GbE、i9-13900H × 3台で70B Q4_K_Mが完走した。

---

## 2026-09-10: ローカル70Bによるリポジトリ監査PoC

### 対象

```text
C:\dev\Last_Beacon-main
```

### 監査入力生成

`node_modules`, `.git`, build成果物、`.env`等を除外し、主要なコード・設定・READMEを収集。

```text
Files found : 21
Prompt chars: 71172
```

### 最初の監査: 32k context

```text
-c 32768
-n 2500
```

Scheduled Taskとして実行。いったんタスク自体は `Ready / Last Result 0` になったものの、出力を確認すると実際にはcontext不足で監査されていなかった。

```text
Error: request (32802 tokens) exceeds the available context size (32768 tokens), try increasing it

[ Prompt: 0.0 t/s | Generation: 0.0 t/s ]
```

入力が32,802 tokensで、32,768 contextを34 tokens超過していた。

### 48k contextで再試行

次の条件へ変更。

```text
-c 49152
-n 2500
```

これにより約13k tokens以上の出力・余白を確保する方針とした。

### このPoCで得た知見

- repo監査では文字数ではなくtoken数でcontext設計する必要がある
- 大きなrepoを丸ごと1プロンプトへ入れるより、分割監査→統合の方が拡張しやすい
- `Last Result: 0` だけではLLM内部エラーを判定できない。標準出力・エラー内容も確認すべき
- 長時間処理はLabOpsから直接起動せずScheduled Task等へ切り離すと安定する
- 夜間バッチ用途では対話速度よりクラスタ全体のジョブスループットが重要

## 次の構想

30台を1つの巨大クラスタにするのではなく、3台 × 10クラスタ程度へ分ける。

```text
repository queue
  ├─ repo01 -> cluster01
  ├─ repo02 -> cluster02
  ├─ ...
  ├─ repo10 -> cluster10
  ├─ repo11 -> first free cluster
  └─ repo12 -> next free cluster
```

各clusterが70Bコード監査を担当し、結果Markdownを生成。将来的にはclone/pull → audit → commit → pushまで自動化する。
