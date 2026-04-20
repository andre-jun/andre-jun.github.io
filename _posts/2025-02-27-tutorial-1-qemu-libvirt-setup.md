---
title: "Tutorial 1 - Configurando ambiente de teste com QEMU e libvirt"
date: 2025-02-27 19:37:45 -0300
categories: [Kernel, Tutoriais]
tags: [linux, arm64, qemu, libvirt, kernel]
---

Este é o primeiro post da matéria MAC0470, Desenvolvimento de Software Livre.
O objetivo deste tutorial foi montar um ambiente de testes usando
QEMU e libvirt para compilar e instalar kernels customizados em uma VM,
para que se possa utilizar e testar implementacoes em diversas arquiteturas diferentes.

O tutorial base utilizado foi o do FLUSP:
[Setting up a test environment for Linux Kernel Dev using QEMU and libvirt](https://flusp.ime.usp.br/kernel/qemu-libvirt-setup/)

## O que o tutorial cobre

- Criar um script `activate.sh` que define variáveis de ambiente e abre uma sessão de shell dedicada
- Baixar uma imagem ARM64 do Debian (nocloud) e redimensionar o `rootfs`
- Criar e registrar uma VM ARM64 no libvirt
- Configurar acesso SSH da máquina host para a VM
- Compartilhamento de arquivos host ↔ VM via `virtiofs`

## O que funcionou sem problemas

A maior parte do tutorial foi tranquila. A configuração do `activate.sh`, o
download da imagem Debian ARM64, a criação da VM com `virt-install` e o acesso
SSH funcionaram sem necessidade de adaptação.

## Problema 1: `virt-filesystems` falhava com erro do supermin

O tutorial instrui a usar o comando abaixo para identificar qual partição da
imagem é o `rootfs`, informação necessária antes de redimensioná-la:

```bash
virt-filesystems --long --human-readable --all --add "${VM_DIR}/base_arm64_img.qcow2"
```

Na minha máquina, o comando falhou com o seguinte erro:

```bash
libguestfs: error: /usr/bin/supermin exited with error status 1.
To see full error messages you may need to enable debugging.
Do:
  export LIBGUESTFS_DEBUG=1 LIBGUESTFS_TRACE=1
and run the command again.
```

O problema é no `supermin`, o componente que o libguestfs usa para montar um
appliance temporário. A solução foi usar o `guestfish` diretamente, que consegue
contornar o problema e listar as partições da imagem:

```bash
sudo guestfish --ro -a "${VM_DIR}/base_arm64_img.qcow2" run : list-partitions
```

A saída mostrou `/dev/sda1` como a partição `rootfs`, permitindo seguir com o
`virt-resize` normalmente:

```bash
virt-resize --expand /dev/sda1 "${VM_DIR}/base_arm64_img.qcow2" "${VM_DIR}/arm64_img.qcow2"
```

## Problema 2: IP da VM mudando entre reinicializações

Por padrão, a rede `default` do libvirt distribui IPs via DHCP, e o endereço
da VM muda toda vez que ela é reiniciada — o que quebra qualquer configuração de SSH que
dependa de IP fixo.

A solução foi associar o MAC address da VM a um IP estático na configuração do
libvirt. Primeiro, descubrindo o MAC address:

```bash
sudo virsh dumpxml arm64 | grep "mac address"
```

Depois editando a rede `default`:

```bash
sudo virsh net-edit default
```

Dentro do bloco `<dhcp>`, adicionando uma entrada `<host>`:

```xml
<host mac="XX:XX:XX:XX:XX:XX" name="arm64" ip="192.168.122.100"/>
```

Foi preciso tambem reiniciar a rede:

```bash
sudo virsh net-destroy default
sudo virsh net-start default
```

A partir daí, a VM sempre recebe o mesmo IP independente de quantas vezes for
reiniciada.

## Resultado

Ambiente funcional com a VM ARM64 Debian acessível via SSH com IP fixo, pronto
para os próximos tutoriais.
