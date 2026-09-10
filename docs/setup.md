# セットアップ手順

このページは、今回のPoCで実際に使った構成とコマンドを整理したものです。

## ディレクトリ

```text
C:\llm\
  llama.cpp\
  tools\
    cmake-4.4.3-windows-x86_64\
  models\
```

## llama.cppの固定コミット

今回の3台はすべて同一コミットへ固定しました。

```text
1945e092030f8668ff93382799502d01490e564d
```

## CMake configure

```powershell
$cmakeExe = "C:\llm\tools\cmake-4.4.3-windows-x86_64\bin\cmake.exe"

Set-Location "C:\llm\llama.cpp"

& $cmakeExe `
  -S . `
  -B build `
  -G "Visual Studio 17 2022" `
  -A x64 `
  -DGGML_RPC=ON `
  -DGGML_CCACHE=OFF
```

設定時に以下を確認しました。

```text
-- Using RPC backend
-- Including RPC backend
```

## 親機で必要ターゲットだけビルド

フルビルドはLabOpsの600秒制限に到達したため、必要なターゲットだけビルドしました。

```powershell
$cmakeExe = "C:\llm\tools\cmake-4.4.3-windows-x86_64\bin\cmake.exe"
Set-Location "C:\llm\llama.cpp"

& $cmakeExe `
    --build build `
    --config Release `
    --target llama-cli ggml-rpc-server `
    --parallel 8
```

生成物:

```text
C:\llm\llama.cpp\build\bin\Release\llama-cli.exe
C:\llm\llama.cpp\build\bin\Release\ggml-rpc-server.exe
```

## 子機でRPC serverをビルド

```powershell
$cmakeExe = "C:\llm\tools\cmake-4.4.3-windows-x86_64\bin\cmake.exe"
Set-Location "C:\llm\llama.cpp"

& $cmakeExe `
  --build build `
  --config Release `
  --target ggml-rpc-server `
  --parallel 8
```

## RPC server

子機R841PC025 / R841PC026でTCP 50052を利用しました。

```text
R841PC025  10.40.112.55:50052
R841PC026  10.40.112.58:50052
```

RPCは実験的で安全なプロトコルではないため、Windows FirewallではR841PC000 (`10.40.112.67`) からのみ許可する構成にしました。

起動引数の例:

```powershell
$rpc = "C:\llm\llama.cpp\build\bin\Release\ggml-rpc-server.exe"

Start-Process `
  -FilePath $rpc `
  -WorkingDirectory "C:\llm\llama.cpp\build\bin\Release" `
  -ArgumentList "-H 0.0.0.0 -p 50052 -t 8"
```

LabOpsでは長時間動作する子プロセスを起動するとジョブが終了しないことがあったため、実運用ではScheduled Task等へ切り離す方が扱いやすいです。

## 親機からRPC疎通確認

```powershell
$pc025 = Test-NetConnection 10.40.112.55 -Port 50052 -InformationLevel Quiet
$pc026 = Test-NetConnection 10.40.112.58 -Port 50052 -InformationLevel Quiet

Write-Host "R841PC025 : $pc025"
Write-Host "R841PC026 : $pc026"
```

結果:

```text
R841PC025 : True
R841PC026 : True
```

## RPC device一覧

```powershell
$cli = "C:\llm\llama.cpp\build\bin\Release\llama-cli.exe"

& $cli `
  --rpc 10.40.112.55:50052,10.40.112.58:50052 `
  --list-devices
```

結果:

```text
Available devices:
 RPC0: 10.40.112.55:50052 (32398 MiB, 23493 MiB free)
 RPC1: 10.40.112.58:50052 (32398 MiB, 26743 MiB free)
```

## HTTPS非対応ビルドの注意

今回のllama.cppビルドではOpenSSL開発ファイルが無かったため、`-hf` によるHugging Face直取得は使えませんでした。

```text
error: HTTPS is not supported
```

そのためモデルは `curl.exe` でローカルへ取得し、`-m` で読み込む方針にしました。

## Gemma 3 1B テスト

モデル:

```text
C:\llm\models\gemma-3-1b-it-Q4_K_M.gguf
```

実行:

```powershell
$cli   = "C:\llm\llama.cpp\build\bin\Release\llama-cli.exe"
$model = "C:\llm\models\gemma-3-1b-it-Q4_K_M.gguf"

& $cli `
  -m $model `
  --rpc 10.40.112.55:50052,10.40.112.58:50052 `
  -ngl 99 `
  -c 2048 `
  -n 32 `
  -st `
  --simple-io `
  -p "日本語で『分散推論テスト成功』とだけ答えてください。"
```

結果:

```text
分散推論テスト成功
[ Prompt: 49.0 t/s | Generation: 16.8 t/s ]
```

## Llama 3.3 70B Q4_K_M

モデル:

```text
C:\llm\models\Llama-3.3-70B-Instruct-Q4_K_M.gguf
```

ファイルサイズ:

```text
39.60 GiB
```

実行例:

```powershell
$cli   = "C:\llm\llama.cpp\build\bin\Release\llama-cli.exe"
$model = "C:\llm\models\Llama-3.3-70B-Instruct-Q4_K_M.gguf"

& $cli `
  -m $model `
  --rpc 10.40.112.55:50052,10.40.112.58:50052 `
  -ngl 54 `
  -t 8 `
  -c 2048 `
  -n 24 `
  -st `
  --simple-io `
  -p "Reply with exactly: 70B distributed inference success"
```

LabOpsはPowerShellを600秒でタイムアウトさせるため、70BはScheduled Taskに切り離して実行しました。

結果:

```text
70B distributed inference success
[ Prompt: 2.1 t/s | Generation: 0.8 t/s ]
```
