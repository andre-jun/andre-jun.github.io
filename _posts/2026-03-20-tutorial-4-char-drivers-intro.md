---
title: "Tutorial 4 - Introdução a character device drivers"
date: 2026-03-20 17:47:09 -0300
categories: [Kernel, Tutoriais]
tags: [linux, kernel, drivers, char-device, file-operations]
---

Quarto tutorial da série. O assunto aqui foi character
device drivers — uma das formas mais fundamentais de um driver expor uma
interface para o userspace no Linux. O tutorial foi concluído sem problemas,
e foi o que mais exigiu atenção conceitual: os passos faziam sentido
individualmente, mas o quadro completo de como tudo se encaixa só ficou mais
claro depois.

O tutorial base utilizado foi o do FLUSP:
[Introduction to Linux kernel Character Device Drivers](https://flusp.ime.usp.br/kernel/char-drivers-intro/)

## O que o tutorial cobre

- O conceito de character device como abstração de dispositivos de I/O sequencial
- Major e minor numbers: como o kernel associa arquivos de dispositivo (`/dev/ttyS0`, etc.) aos drivers correspondentes
- A struct `file_operations` e como um driver implementa chamadas de sistema (`open`, `release`, `read`, `write`)
- Alocação dinâmica de major number com `alloc_chrdev_region()`
- Criação manual de device nodes com `mknod`
- Implementação e teste do driver `simple_char` de exemplo

## Sobre não entender tudo enquanto fazia

Esse tutorial tem uma densidade conceitual maior que os anteriores. Cada parte
— números de dispositivo, `file_operations`, `cdev`, device nodes — faz sentido
isoladamente, mas a conexão entre elas (por que o kernel sabe qual função chamar
quando você faz `read()` num arquivo em `/dev/`) só fica clara quando você
enxerga o fluxo completo.

Isso é algo que percebi ser normal seguindo tutoriais: você
segue os passos, o resultado funciona, e o entendimento profundo vem depois —
seja relendo, seja trabalhando em cima disso nos próximos tutoriais.

## Resultado

Driver `simple_char` compilado, carregado na VM e testado com sucesso. O device
node foi criado manualmente com `mknod` e as operações de leitura e escrita
funcionaram conforme esperado.
