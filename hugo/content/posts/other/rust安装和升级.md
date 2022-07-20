---
title: rust安装和升级
<!-- description: 这是一个副标题 -->
date: 2020-09-09
slug: install-and-upgrade-rust
categories:
    - other

tags:
    - other
    
---

## 安装 `rustup`

使用 `rustup` 安装 `rust` ，安装 `rustup`:

```shell
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

在 Rust 开发环境中，所有工具都安装在 `~/.cargo/bin` 目录中，您可以在这里找到包括 `rustc`、`cargo` 和 `rustup` 在内的 Rust 工具链。

## 查看 rust 版本

```shell
rustc --version
```

## 卸载

```shell
rustup self uninstall
```

## 升级

```shell
rustup update
```
