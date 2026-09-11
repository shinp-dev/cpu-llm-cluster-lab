# CPU LLM Cluster Lab

GPUを使わず、複数台のWindows PCを `llama.cpp` のRPCバックエンドで束ね、大規模LLMをCPU分散実行する実証実験の記録です。

## 実証環境

| Role | Host | IP | CPU | RAM | GPU |
|---|---|---|---|---|---|
| Parent / controller | R841PC000 | 10.40.112.67 | Intel Core i9-13900H | 約32GB | なし |
| RPC worker | R841PC025 | 10.40.112.55 | Intel Core i9-13900H | 約32GB | なし |
| RPC worker | R841PC026 | 10.40.112.58 | Intel Core i9-13900H | 約32GB | なし |

ネットワークは1GbE。操作は独自のLabOpsからPowerShellを遠隔実行して行いました。

## ソフトウェア

- Windows
- llama.cpp
- llama.cpp commit: `1945e092030f8668ff93382799502d01490e564d`
- CMake 4.4.3 portable
- Visual Studio 2022 Build Tools
- RPC port: `50052/tcp`

## ここまでの結果

### 1. RPC worker認識

親機R841PC000から以下を確認。

```text
Available devices:
 RPC0: 10.40.112.55:50052 (32398 MiB, 23493 MiB free)
 RPC1: 10.40.112.58:50052 (32398 MiB, 26743 MiB free)
```

### 2. 小型モデルで分散推論

`gemma-3-1b-it-Q4_K_M.gguf` で分散推論に成功。

```text
分散推論テスト成功

[ Prompt: 49.0 t/s | Generation: 16.8 t/s ]
```

### 3. Llama 3.3 70B Q4_K_M

`Llama-3.3-70B-Instruct-Q4_K_M.gguf`（約39.60GiB）を3台CPU構成で実行。

```text
70B distributed inference success

[ Prompt: 2.1 t/s | Generation: 0.8 t/s ]
```

GPUなし、1GbE、i9-13900H × 3台で70Bが実際に完走しました。

### 4. 実リポジトリ監査を完走

`C:\dev\Last_Beacon-main` の主要21ファイルをまとめ、約32,802 tokenの入力を48k contextで監査。

```text
Start      : 2026-09-10 17:08:44
Output end : 2026-09-11 03:47:18
Elapsed    : 約10時間38分

Prompt     : 1.1 t/s
Generation : 0.1 t/s
```

結果Markdownの生成まで完走した。

ただし、一括長文監査では以下の課題も確認。

- prefillが非常に重い
- 長文生成が遅い
- severity判定が甘い
- 根拠の弱い一般論が混ざる
- worker電源断で長時間ジョブが全損する

次段階では token-aware chunking / checkpoint / retry / focused audit を導入する。

## 想定用途

最終的には30台程度のPCを `3台 × 10クラスタ` に分割し、夜間バッチで複数リポジトリを並列監査する構成を想定しています。

```text
GitHub / local repository
        ↓
   audit queue
        ↓
10 x 70B clusters
        ↓
token-aware chunk audit
        ↓
structured findings
        ↓
final AUDIT.md
        ↓
commit / push
```

1本の推論を30台へ細分化して高速化するより、3台クラスタを複数作り、独立ジョブを並列処理する方針です。

## ドキュメント

- [セットアップ手順](docs/setup.md)
- [実験ログ](docs/experiment-log.md)
- [リポジトリ監査PoC](docs/repository-audit-poc.md)
- [Lessons Learned](docs/lessons-learned.md)

## 注意

`llama.cpp` のRPC機能は実験的な機能です。RPCサーバーを外部ネットワークへ公開しないでください。本PoCではWindows Firewallで親機R841PC000からの接続だけを許可しています。
