---
title: Tryout Deno
date: 2022-09-04T23:41:50+09:00
tags: [deno,typescript]
---
Deno runtimeの場合V8, Rust, Tokioで作られたJavascript, Typescript, WebAssembly Runtimeです。 今回は簡単な例を確認してnodejsとどう違うのか確認する時間を作ってみます。

## 環境

今回のpostで使ったlocal環境は下記のような環境です。

```text
Machine: Macbook Pro (16-inchim, 2021), Apple M1 Pro
OS: MacOS Monterey 12.6.3
```

## Denoインストール

[Deno installation](https://deno.land/manual@v1.36.3/getting_started/installation)を参考したら、下記のcommandで簡単にインストールができます。

```sh
curl -fsSL https://deno.land/x/install/install.sh | sh
```

deno binaryが自動でインストールされ、下記のコマンドでdeno binaryをPATHに登録します。

```sh
cat <<EOF >>$HOME/.zshrc
# Deno
export DENO_INSTALL="/Users/user/.deno"
export PATH="\$DENO_INSTALL/bin:\$PATH"
EOF
source $HOME/.zshrc
```

## Deno binaryインストール確認

```sh
$ deno --version
deno 1.36.3 (release, aarch64-apple-darwin)
v8 11.6.189.12
typescript 5.1.6
```

## Deno vscode pluginインストール

vscodeをIDEで使う開発者のため[deno plugin](https://marketplace.visualstudio.com/items?itemName=denoland.vscode-deno)を提供しています。`install`ボタンを押してインストールします。

![deno vscode plugin page](images/vscode-deno-plugin.webp)

そしてvscodeのdeno関連のworkspaceでdeno pluginをenableするため下記のファイルを生成します。 (既存のファイルがあったらjsonに該当key/valueを追加します。)

```json
# path: .vscode/settings.json
{
  "deno.enable": true,
  "deno.lint": true,
  "editor.formatOnSave": true,
  "[typescript]": { "editor.defaultFormatter": "denoland.vscode-deno" }
}
```

## Hello, Worldの例題実行

下記のように`tryout-deno.ts`typescript fileを作って下記のコードをコピーして入れます。

```typescript
# tryout-deno.ts
console.log('Hello, World!')
```

上のコードは下記のように動作させると`Hello, World！`が出ます。 typescript関連libraryなどをインストールする必要がなくdeno binaryでサポートします。

```sh
$ deno run tryout-deno.ts 
Hello, World!
```

## deno project生成

deno binaryを使ってdenoのsample codeを生成して生成されたfilesについて簡単に説明します。 下記のコマンドを入力すると簡単に生成することができます。

```sh
deno init
```

そしたら、下記のように4つのファイルが生成されます。

```sh
$ tree
.
├── deno.jsonc
├── main.ts
├── main_bench.ts
└── main_test.ts

1 directory, 4 files
```

1. deno.jsonc:`package.json`と同じくdenoの設定ファイルです。詳細は[こちら](https://deno.land/manual@v1.36.4/getting_started/configuration_file)を参考してください。
2. main.ts: 自動生成された例では簡単なadd関数と例があります。上の`deno.jsonc`のtaskで該当`main.ts`を実行できる`dev`taskが例として記載されています。
3. main\_bench.ts: denoでは`deno bench`を使って [benchmarkをすることができるAPIとtool](https://deno.land/manual@v1.36.4/tools/benchmarker)を提供しています。現在の例では`main.ts`のadd関数に対するbenchmarkの例があります。
4. main\_test.ts: denoでは`deno test`を通して [testのためのAPIとtool](https://deno.land/manual@v1.36.4/basics/testing#testing)を提供しています。現在の例では`main.ts`のadd関数に対するtest例があります。

## Conclusion

deno binaryをインストールして簡単なsample codeを生成した後、生成されたファイルについて簡単に見てみました。 既存のnodeよりtypescriptを基本的にサポートし、packageも別にインストールしなくてもcodeに直接指定することができるのが新鮮でした。次はdenoを使ってbackend serverを作成してみます。
