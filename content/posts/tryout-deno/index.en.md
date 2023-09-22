---
title: Tryout Deno
date: 2022-09-04T23:41:50+09:00
tags: [deno,typescript]
---

Deno runtime is a Javascript, Typescript, and WebAssembly Runtime made with V8, Rust, and Tokio. Let's take a look at a simple example and see how it compares to nodejs.

## The Environment

The local environment used in this post is as follows.

```text
Machine: Macbook Pro (16-inchim, 2021), Apple M1 Pro
OS: MacOS Monterey 12.6.3
```

## Installing Deno

Refer to [Deno installation](https://deno.land/manual@v1.36.3/getting_started/installation) and execute the command below to install [Deno](https://deno.land/manual@v1.36.3/getting_started/installation).

```sh
curl -fsSL https://deno.land/x/install/install.sh | sh
```

The deno binary will be installed automatically, and execute the command below to register the deno binary in the PATH.

```sh
cat <<EOF >>$HOME/.zshrc
# Deno
export DENO_INSTALL="/Users/user/.deno"
export PATH="\$DENO_INSTALL/bin:\$PATH"
EOF
source $HOME/.zshrc
```

## Verify the installation of the Deno binary

```sh
$ deno --version
deno 1.36.3 (release, aarch64-apple-darwin)
v8 11.6.189.12
typescript 5.1.6
```

## Install the Deno vscode plugin

For developers who use vscode as their IDE, we provide the [deno plugin](https://marketplace.visualstudio.com/items?itemName=denoland.vscode-deno). Click the `install` button to install it.

![deno vscode plugin page](images/vscode-deno-plugin.webp)

Then, create the file below to enable the deno plugin in the deno-related workspace of vscode. (If you already have the file, add the key/values to the json).

```json
# path: .vscode/settings.json
{
  "deno.enable": true,
  "deno.lint": true,
  "editor.formatOnSave": true,
  "[typescript]": { "editor.defaultFormatter": "denoland.vscode-deno" }
}
```

## Run the Hello, World example

Create a `tryout-deno.ts` typescript file like below and copy and paste the code below into it.

```typescript
# tryout-deno.ts
console.log('Hello, World!')
```

If you run the above code like below, `Hello, World!`will be displayed. You don't need to install any typescript related libraries, it is supported by deno binary.

```sh
$ deno run tryout-deno.ts 
If you run the above code, Hello, World!
```

## Create a deno project

Let's create a sample code of deno using deno binary and give a brief explanation of the created files. Enter the command below to create it.

```sh
deno init
```

This will create 4 files like below.

```sh
$ tree
.
├── deno.jsonc
├── main.ts
├── main_bench.ts
└── main_test.ts

1 directory, 4 files
```

1. deno.jsonc: The configuration file for deno, `like package.json`. For more information, please see [here](https://deno.land/manual@v1.36.4/getting_started/configuration_file).
2. main.ts: The auto-generated example has a simple add function and examples. In the task `in deno.jsonc`above, a `dev` task is specified as an example that can perform that `main.ts`.
3. main\_bench.ts: deno provides [APIs and tools to benchmark](https://deno.land/manual@v1.36.4/tools/benchmarker)through `deno bench`. In the current example, there is an example benchmark for the add function `in main.ts`.
4. main\_test.ts: deno provides [APIs and tools for testing](https://deno.land/manual@v1.36.4/basics/testing#testing)through `deno test`. In the current example book, there is a test example for the add function `in main.ts`.

## Conclusion

After installing the deno binary and generating a simple sample code, I took a quick look at the generated files. I felt that it was different to have native support for typescript than node, and to be able to specify it directly in the code without having to install a package. Next time, I will try to write a backend server using deno.
