---
title: "Tutorial 8 - Experimento com com o iio_dummy"
date: 2026-04-01 10:11:43 -0300
categories: [Kernel, Tutoriais]
tags: [linux, kernel, iio, drivers, sysfs, configfs, módulos]
---

Oitavo tutorial, complemento prático do anterior. Depois de estudar a anatomia
do `iio_simple_dummy`, aqui o objetivo é colocar o módulo pra rodar e
interagir com ele de dentro da VM.

O tutorial base utilizado foi o do FLUSP:
[IIO Dummy module Experiment One: Play with iio\_dummy](https://flusp.ime.usp.br/iio/experiment-one-iio-dummy/)

## O que o tutorial cobre

- Habilitar o `iio_dummy` no `.config` via `nconfig`
  (`CONFIG_IIO_SIMPLE_DUMMY`, `CONFIG_IIO_DUMMY_EVGEN`, etc.)
- Compilar apenas o subdiretório do driver com `make M=drivers/iio/dummy`
- Carregar o módulo com `modprobe iio_dummy` e verificar com `lsmod` e `modinfo`
- Explorar os arquivos expostos em `/sys/bus/iio/devices/` após o carregamento
- Montar o `configfs` e criar um dispositivo dummy virtual via
  `mkdir /mnt/iio_experiments/iio/devices/dummy/meu_dispositivo`
- Adicionar canais de bússola de 3 eixos ao `iio_simple_dummy` como exercício
  de modificação de driver real

## Conceito relevante

O `configfs` é um sistema de arquivos do kernel que permite criar e configurar
objetos do kernel a partir do userspace via operações de diretório. No contexto
do `iio_dummy`, ele é usado para instanciar dispositivos virtuais em tempo de
execução sem precisar recompilar ou recarregar o módulo — o que o torna útil
para testes.

## Resultado

Tutorial concluído sem problemas. A parte de adicionar os canais de
bússola é o primeiro exercício de modificação de código IIO de verdade da
série, e funcionou como esperado após rebuild e reinstalação dos módulos.
