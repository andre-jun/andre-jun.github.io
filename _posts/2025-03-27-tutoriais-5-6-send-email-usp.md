---
title: "Tutoriais 5 e 6 - Enviando patches por email com git e email USP"
date: 2025-03-27 19:16:56 -0300
categories: [Kernel, Tutoriais]
tags: [linux, kernel, git, send-email, kw, patches, docker, oauth]
---

Esses dois tutoriais andam juntos: o Tutorial 5 cobre o fluxo geral de envio
de patches por email com `git send-email`, e o Tutorial 6 é uma extensão
específica para quem usa email USP, que tem algumas particularidades de
autenticação. O recomendado era ler o 5 antes de fazer o 6, e foi exatamente
assim que fiz.

Tutorial 5:
[Sending patches by email with git](https://flusp.ime.usp.br/git/sending-patches-by-email-with-git/)

Tutorial 6:
[Sending patches with git and a USP email](https://flusp.ime.usp.br/git/sending-patches-with-git-and-a-usp-email/)

## Por que patches por email?

O desenvolvimento do kernel Linux (e de vários outros projetos de software
livre) não usa Pull Requests do GitHub. Contribuições são enviadas como patches
por email para mailing lists. O `git send-email` automatiza a conversão de
commits em emails formatados corretamente, evitando que clientes de email
comuns quebrem o código com HTML ou conversões de espaçamento.

## Tutorial 5: configuração geral do `git send-email`

O Tutorial 5 cobre a configuração do `git send-email` para qualquer conta de
email, com exemplos para Gmail. Os principais passos são configurar o servidor
SMTP, porta, usuário e criptografia no `.gitconfig`:

```bash
git config --global sendemail.smtpencryption tls
git config --global sendemail.smtpserver "smtp.gmail.com"
git config --global sendemail.smtpuser "seu@email.com"
git config --global sendemail.smtpserverport "587"
```

O tutorial também cobre o uso de `--annotate`, `--cover-letter`, `--thread` e
`--dry-run`, além de versionamento de patches com `-v2`, `-v3` etc.

## Tutorial 6: o problema com email USP

Emails USP são contas Gmail institucionais, mas com uma restrição importante:
não é possível habilitar 2FA nelas, o que impede a criação de App Passwords —
o método padrão para autenticar serviços externos no Gmail desde janeiro de
2025. O método alternativo de "Less Secure Apps" também foi bloqueado pelos
administradores da USP após incidentes de segurança em 2025.

A solução é usar um proxy de email que implementa OAuth 2.0, o protocolo que
delega a autenticação ao próprio Google sem expor senha. O proxy utilizado é o
[email-oauth2-proxy](https://github.com/simonrob/email-oauth2-proxy/), rodando
dentro de um container Docker via um repositório fornecido pelo tutorial:

```bash
git clone https://github.com/davidbtadokoro/emailproxy-container.git
```

O fluxo é:
1. Preencher `emailproxy.config` com o email USP e os tokens OAuth (client ID
   e secret), obtidos em um pad da matéria
2. Subir o container com `docker compose up --build`
3. Entrar no container e iniciar o proxy com `emailproxy --no-gui --external-auth`
4. Configurar o `kw send-patch` apontando para o proxy local (`127.0.0.1:2587`)

```bash
kw send-patch --setup --name 'nome'
kw send-patch --setup --email 'email@usp.br'
kw send-patch --setup --smtpuser 'email@usp.br'
kw send-patch --setup --smtpserver '127.0.0.1'
kw send-patch --setup --smtpserverport '2587'
```

## Sobre o ambiente do Gabriel

O container Docker do proxy de email precisa rodar apenas uma vez na máquina
host — ele fica escutando localmente na porta `2587`. Já a configuração do
`kw send-patch` fica armazenada no `.git/config` local de cada repositório,
então o Gabriel configurou o lado dele de forma independente dentro da própria
pasta do projeto, sem necessidade de subir um segundo container.

## Resultado

Setup funcional. O envio de um patch de teste confirmou que o proxy OAuth estava
funcionando corretamente com o email USP e que o `kw send-patch` estava
apontando para o proxy local.

