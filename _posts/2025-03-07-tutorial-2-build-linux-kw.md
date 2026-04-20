---
title: "Tutorial 2 - Compilando e bootando um kernel Linux customizado com kw"
date: 2025-03-07 19:12:03 -0300
categories: [Kernel, Tutoriais]
tags: [linux, arm64, kw, kworkflow, kernel, cross-compile]
---

Continuação do Tutorial 1. O objetivo aqui foi compilar o kernel Linux a partir
do código fonte para ARM64 e bootá-lo na VM, usando o
[kworkflow (`kw`)](https://kworkflow.org/) para automatizar boa parte do processo.

O tutorial base utilizado foi o do FLUSP:
[Building and booting a custom Linux kernel for ARM using kw](https://flusp.ime.usp.br/kernel/build-linux-for-arm-kw/)

## O que o tutorial cobre

- Instalação do `kw`
- Clone da árvore IIO do kernel (`jic23/iio`, branch `testing`)
- Configuração do `kw` no contexto local da árvore IIO (remote SSH, arquitetura, cross-compiler)
- Criação de um `.config` enxuto com `defconfig` + `localmodconfig`
- Compilação do kernel com `kw build`
- Instalação dos módulos e boot do kernel customizado na VM

## Conceitos relevantes

O `kw` funciona de forma análoga ao `git` em termos de configuração: existe uma
configuração global e uma local por árvore, inicializada com `kw init` dentro
do diretório da árvore. Todos os comandos rodados dentro dessa árvore respeitam
as configurações locais.

O `.config` gerado via `localmodconfig` merece destaque: em vez de compilar
todos os módulos possíveis para ARM64, o comando usa a lista de módulos
atualmente carregados na VM (`vm_mod_list`) para gerar um `.config` mínimo.
Isso reduz drasticamente o tempo de compilação — relevante dado o tamanho da
árvore do kernel.


## Resultado

Tutorial concluído sem intercorrências. Ao final, a VM estava rodando um kernel
compilado localmente a partir da árvore IIO, com os módulos instalados via
`kw deploy --modules`. O fluxo completo de build e deploy funcionou conforme
descrito no tutorial.
