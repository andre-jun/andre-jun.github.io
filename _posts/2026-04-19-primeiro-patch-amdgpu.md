---
title: "Primeiro patch no kernel Linux: Deduplicando código no driver amdgpu"
date: 2026-04-19 12:12:45 -0300
categories: [Kernel, Patches]
tags: [kernel, linux, amdgpu, drm, patch, contribuição]
---

Neste post vou documentar o primeiro patch que enviei para o kernel Linux,
feito em conjunto com Gabriel Dimant e Guilherme Gabriel na
disciplina MAC0470, Desenvolvimento de Software Livre da USP.

## O subsistema

O driver escolhido foi o **amdgpu**, o driver de GPU AMD presente no kernel
Linux, dentro do subsistema DRM (`drivers/gpu/drm/amd/amdgpu/`). A escolha
foi sugerida pelo professor, que disponibilizou várias sugestões de patches
para escolhermos.

## O que encontramos

Navegando pelo código, identificamos que os arquivos `gmc_v10_0.c` e
`gmc_v11_0.c` contêm duas funções idênticas linha a linha:

- `gmc_v10_0_get_vm_pde` / `gmc_v11_0_get_vm_pde`
- `gmc_v10_0_get_vm_pte` / `gmc_v11_0_get_vm_pte`

Essas funções são responsáveis por configurar entradas nas tabelas de páginas
da GPU (_Page Directory Entries_ e _Page Table Entries_) para as gerações
NAVI10 (v10) e RDNA3 (v11) do hardware AMD.

A duplicação provavelmente surgiu porque `gmc_v11_0.c` foi criado a partir de
uma cópia de `gmc_v10_0.c` como base, e essas funções específicas nunca
precisaram divergir entre as duas gerações.

## Por que mover para `amdgpu_gmc.c` e não fazer v11 chamar v10

A primeira ideia foi simplesmente fazer as funções da v11 chamarem as da v10.
Mas isso introduziria uma dependência semântica falsa: implicaria que v11
depende da implementação de v10, o que não é verdade — são hardwares
independentes que coincidem nesse comportamento.

Além disso, se no futuro alguém modificar `gmc_v10_0_get_vm_pte` por razões
específicas do hardware v10, quebraria a v11 silenciosamente.

A solução mais correta foi mover a implementação para `amdgpu_gmc.c`, o arquivo
de código compartilhado entre as versões do GMC, com os nomes
`amdgpu_gmc_nv_get_vm_pde` e `amdgpu_gmc_nv_get_vm_pte`. As structs de
funções de v10 e v11 passaram a apontar diretamente para essas
implementações comuns.

## Dificuldades no caminho

**Conflito de nomes com macros existentes:** ao tentar nomear as funções
como `amdgpu_gmc_get_vm_pde` e `amdgpu_gmc_get_vm_pte`, a compilação falhou
porque já existiam macros com esses nomes no header `amdgpu_gmc.h` para o
dispatch via ponteiro de função. A solução foi usar o prefixo `nv` para
deixar claro que são helpers específicos para hardware NV10/NV11.

**Include faltando:** as constantes `MTYPE_NC`, `MTYPE_WC`, `MTYPE_CC` e
`MTYPE_UC` usadas na função `get_vm_pte` estavam definidas em
`navi10_enum.h`, que `gmc_v10_0.c` já incluía mas `amdgpu_gmc.c` não. Foi
necessário adicionar o include manualmente.

**Comentários de formato de hardware:** os comentários acima das funções
originais descreviam o formato de bits das PTEs e PDEs de cada geração, e
eram diferentes entre v10 e v11. Como são documentações específicas de
hardware, foram mantidos nos arquivos originais junto às structs de funções,
e não movidos para o arquivo comum.

**Warning do `checkpatch.pl`:** o script apontou um `BUG_ON` na função
movida, sugerindo o uso de `WARN_ON_ONCE`. Como era código preexistente
apenas relocado, optei por documentar isso no commit message em vez de
alterar o comportamento.

## Divisão do trabalho

O patch foi desenvolvido em conjunto com Gabriel Dimant e Guilherme Gabriel.
A discussão sobre a abordagem correta e a implementação foram feitas em grupo.
O envio ficou comigo, com os três listados no commit via `Co-developed-by`.

## O patch

A mensagem de commit ficou assim:

```
drm/amdgpu: unify gmc v10 and v11 get_vm_pde and get_vm_pte into common helpers

gmc_v10_0_get_vm_pde, gmc_v10_0_get_vm_pte and their v11 counterparts
are identical. Move the shared implementation to amdgpu_gmc.c as
amdgpu_gmc_nv_get_vm_pde and amdgpu_gmc_nv_get_vm_pte, and update both
gmc_v10_0 and gmc_v11_0 to use the common helpers.

No functional changes intended. BUG_ON preserved from original
gmc_v10_0 and gmc_v11_0 implementations.

Signed-off-by: Andre Jun Hirata <andrejhirata@usp.br>
Co-developed-by: Gabriel Dimant <gabriel.dimant@usp.br>
Signed-off-by: Gabriel Dimant <gabriel.dimant@usp.br>
Co-developed-by: Guilherme Gabriel <guilhermesangabriel@usp.br>
Signed-off-by: Guilherme Gabriel <guilhermesangabriel@usp.br>
```

## Envio

O patch foi enviado aos mantenedores e para a mailing list do subsistema amdgpu:

- **Maintainers:** Alex Deucher e Christian König
- **Lista:** `amd-gfx@lists.freedesktop.org`

o patch pode ser encontrado no [lore.kernel](https://lore.kernel.org/amd-gfx/20260418201545.20673-1-andrejhirata@usp.br/)

## Resultado

O patch foi rejeitado pelo mantenedor Christian König com a seguinte resposta:

```
Well again this is not something we want to do.
Those functions are intentionally separated.

Regards,
Christian.
```

Ou seja, mesmo sendo idênticas hoje, as funções são mantidas separadas
intencionalmente. Foi um aprendizado importante — código duplicado nem sempre é um
problema a ser resolvido, especialmente quando a separação é uma decisão
arquitetural consciente dos mantenedores.

Isso levanta a questão: como saber, de fora, se uma duplicação é um descuido ou uma decisão? Às vezes a única forma de descobrir é enviando o patch.
