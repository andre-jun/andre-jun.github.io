---
title: "Intermission - Preparando o ambiente para um segundo usuário"
date: 2026-03-07 15:43:12 -0300
categories: [Kernel, Intermission]
tags: [linux, qemu, libvirt, tailscale, setup]
---

Este post é um intermission — não é um tutorial da série oficial, mas documenta
uma adaptação necessária antes de continuar. A matéria é feita em trio com o
Gabriel Dimant e o Guilherme Gabriel, infelizmente o Gabriel Dimant é um usuario de `Windows` e, 
como eu ja teria que acessar o meu Desktop por meio de ssh na aula ofereci o mesmo para meu colega.
Para progressao individual nos tutoriais o melhor é cada um ter seu próprio ambiente
isolado na mesma máquina host em vez de compartilhar arquivos e VMs. Isso
evita conflitos e deixa claro o que pertence a cada um durante todo o
desenvolvimento.

## O problema: tudo estava no singular

Depois de completar o Tutorial 1 para mim, percebi que precisaria fazer o mesmo
para o Gabriel — mas tudo que eu havia criado estava nomeado de forma genérica
ou pensando em um único usuário: o diretório `/home/lk_dev`, a VM `arm64`, os
scripts. Antes de replicar o ambiente, a decisão foi renomear tudo para tornar
a distinção explícita.

## Renomeando o ambiente existente

O diretório de trabalho passou de `/home/lk_dev` para `/home/lk_jun`, e a VM
foi renomeada de `arm64` para `arm64-jun`. Os scripts `activate.sh` e todas as
variáveis de ambiente internas foram atualizados para refletir os novos caminhos:

```bash
# Variáveis no activate.sh do Jun
export LK_DEV_DIR='/home/lk_jun'
export VM_DIR="${LK_DEV_DIR}/vm"
```

A VM também foi redefinida com um MAC address fixo associado a um IP estático
dedicado na rede `default` do libvirt, como descrito no post anterior.

## Criando o ambiente do Gabriel

Com o ambiente próprio bem delimitado, foi só repetir o Tutorial 1 inteiro para
o Gabriel. O resultado foi uma estrutura paralela e independente:

- Diretório de trabalho: `/home/lk_gab`
- VM: `arm64-gab`
- Scripts, imagens de disco, árvore do kernel e arquivos de boot todos separados
  dentro de `/home/lk_gab`

```bash
# Variáveis no activate.sh do Gabriel
export LK_DEV_DIR='/home/lk_gab'
export VM_DIR="${LK_DEV_DIR}/vm"
```

A VM do Gabriel também recebeu seu próprio MAC address e IP fixo na rede do
libvirt, evitando qualquer conflito com a minha.

Ao final, avisei o Gabriel que o ambiente dele estava pronto, que suas coisas
estavam em `/home/lk_gab` e que a VM dele era a `arm64-gab` — ele poderia
continuar os próximos tutoriais a partir daí.

## Acesso remoto com Tailscale

Como a máquina host (`DontFreeze`) fica fisicamente no meu quarto, tanto eu
quanto o Gabriel precisamos de uma forma de acessá-la de fora — seja em aula,
seja em casa. A solução foi instalar o Tailscale no host.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

O Tailscale cria uma VPN mesh entre os dispositivos autorizados, atribuindo um
IP fixo privado para cada um. Com isso:

- Eu consigo acessar a `DontFreeze` pelo celular durante as aulas
- O Gabriel consegue acessar de casa ou da faculdade

O acesso é feito via SSH normalmente, usando o IP do Tailscale no lugar do IP
local:

```bash
ssh jun@<tailscale-ip-da-DontFreeze>
```

De dentro da máquina, cada um acessa sua própria VM via SSH pelo IP fixo
configurado no libvirt.

## Resultado

Dois ambientes completamente independentes na mesma máquina, com acesso remoto
funcional via Tailscale. O Gabriel pode seguir os próximos tutoriais no próprio
espaço sem interferir no meu, e vice-versa.
