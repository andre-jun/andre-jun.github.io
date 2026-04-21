---
title: "Tutorial 3 - Introdução a módulos e configuração de build do kernel"
date: 2026-03-14 18:43:18 -0300
categories: [Kernel, Tutoriais]
tags: [linux, arm64, kbuild, kconfig, módulos, kernel]
---

Terceiro tutorial da série. O foco aqui saiu do ambiente e da compilação do
kernel em si para o sistema de build e módulos: como o kernel decide o que
compilar, como adicionar código novo a essa estrutura, e como carregar e
descarregar módulos dinamicamente.

O tutorial base utilizado foi o do FLUSP:
[Introduction to Linux kernel build configuration and modules](https://flusp.ime.usp.br/kernel/modules-intro/)

## O que o tutorial cobre

- Criação de um módulo simples (`simple_mod.c`) em `drivers/misc/`
- Criação de um símbolo de configuração `Kconfig` para o módulo (`tristate`, `default`, `depends on`)
- Adição do módulo ao `Makefile` do diretório via `obj-$(CONFIG_SIMPLE_MOD)`
- Habilitação do módulo via `menuconfig` e rebuild com `kw build`
- Instalação dos módulos na VM com `guestmount` + `modules_install`
- Uso de `insmod`, `rmmod`, `modprobe` e `modinfo` dentro da VM
- Verificação dos logs do kernel com `dmesg`

## Conceitos relevantes

O sistema de build do kernel (`kbuild`) usa os arquivos `Kconfig` de cada
diretório para definir símbolos de configuração, que por sua vez controlam o
que é compilado. Um símbolo do tipo `tristate` pode assumir três valores: `y`
(built-in no kernel), `m` (compilado como módulo separado) ou `n` (não
compilado). Esses valores ficam armazenados no `.config`.

## Resultado

Tutorial concluído sem intercorrências. O `simple_mod` foi compilado, instalado
na VM, carregado com `insmod` e as mensagens `"Hello world"` e `"Goodbye world"`
apareceram corretamente no `dmesg`.

