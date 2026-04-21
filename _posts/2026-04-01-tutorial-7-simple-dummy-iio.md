---
title: "Tutorial 7 - Anatomia do driver iio_simple_dummy"
date: 2026-04-01 10:11:43 -0300
categories: [Kernel, Tutoriais]
tags: [linux, kernel, iio, drivers, subsistema]
---

Sétimo tutorial da série, agora entrando de fato no subsistema IIO. O foco
aqui é leitura e análise de código: o tutorial faz uma dissecção do driver
`iio_simple_dummy`, que existe no kernel justamente como um driver de referência
didático para quem quer entender como drivers IIO são estruturados.

O tutorial base utilizado foi o do FLUSP:
[The iio\_simple\_dummy Anatomy](https://flusp.ime.usp.br/iio/iio-dummy-anatomy/)

## O que o tutorial cobre

- O conceito de canal IIO (`iio_chan_spec`) e como um único dispositivo pode
  ter múltiplos canais (exemplo: acelerômetro com X, Y e Z)
- Os campos da struct `iio_chan_spec`: `.type`, `.indexed`, `.channel`,
  `.info_mask_separate`, `.info_mask_shared_by_dir`, `.scan_index`, `.scan_type`
- A diferença entre atributos separados por canal (`info_mask_separate`) e
  atributos compartilhados por direção (`info_mask_shared_by_dir`)
- Como eventos são registrados via `.event_spec` e habilitados condicionalmente
  via `CONFIG_IIO_SIMPLE_DUMMY_EVENTS`
- A estrutura geral de inicialização e remoção de um driver IIO

## Conceito relevante

A `iio_chan_spec` é massivamente configurável por design: o mesmo struct é
usado para representar desde um canal de tensão de um ADC até um eixo de um
acelerômetro. Os campos de máscara de informação (`info_mask_*`) controlam
quais atributos ficam expostos para o userspace via sysfs — o que o driver
"sabe" sobre cada canal.

## Resultado

Tutorial concluído sem intercorrências. Leitura densa mas bem comentada —
o `iio_simple_dummy` tem comentários extensos no próprio código que ajudam
bastante a acompanhar o tutorial.
