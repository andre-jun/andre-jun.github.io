---
title: "Contribuindo para o KDE Plasma — Mouse KCM"
date: 2026-06-08 21:23:14 -0300
categories: [KDE, Patches]
tags: [KDE, linux, onboarding, patch, contribuição]
---

## Escolhendo um projeto

Este é um post da matéria MAC0470, Desenvolvimento de Software Livre, acerca da segunda fase da disciplina. Nessa fase a ideia era contribuir para um projeto open source curado pelo professor e monitores. O grupo optou por contribuir para o KDE, e nosso professor nos colocou em contato com o Farid, um membro da equipe do KDE. Na primeira reunião ele nos deu uma visão geral do que é o KDE e da escala do projeto — não esperava descobrir que o software do KDE está presente no sistema de infoentretenimento de carros de 2025, por exemplo.

Ele nos apresentou o ambiente de desenvolvimento: KDE Builder, KDE Invent, as rooms relevantes no Matrix. Depois apresentou algumas propostas de contribuição: melhorias no Mouse KCM, um teclado virtual, drivers para mesas digitalizadoras, entre outras.

Após essa reunião fomos ver os materiais e decidimos trabalhar no Mouse KCM. A ideia de melhorar a experiência de mouses gaming no KDE Plasma pareceu tecnicamente interessante e concretamente útil.

## Entendendo a tarefa

Em uma segunda reunião o Farid explicou a proposta com mais detalhes. A ideia geral era trazer funcionalidades similares às que o libratbag/Piper já oferecem — remapeamento de botões, configuração de DPI, controle de LED, ilustrações do mouse — mas integradas nativamente na página de configurações de mouse do Plasma.

O Mouse KCM atualmente é assim:



![KCM do mouse atual 1](/assets/img/posts/kde-mouse-kcm/mouse-kcm-current-1.png)

*O Mouse KCM atual no KDE Plasma*

E esse é o Piper, o app GTK que acompanha o libratbag:



![Piper resolution](/assets/img/posts/kde-mouse-kcm/piper-resolutionpage.png)
![Piper buttons](/assets/img/posts/kde-mouse-kcm/piper-buttonpage.png)
![Piper led](/assets/img/posts/kde-mouse-kcm/piper-ledpage.png)


*Piper — o frontend GTK do libratbag*

O objetivo é trazer algo mais próximo da experiência do Piper nativamente no Plasma, sem necessariamente depender do libratbag para tudo.

As tarefas iniciais foram: explorar o repositório do Mouse KCM, olhar o código do libratbag, e entrar na room do KWin no Matrix para acompanhar as discussões relevantes.

## Explorando o código

Vasculhando o backend do KWin, uma das primeiras coisas que notei foi que os IDs de `vendor` e `product` já estão expostos pela API D-Bus do InputDevices. Foi uma descoberta útil porque significa que dá para identificar modelos de mouse sem precisar de nenhuma infraestrutura adicional.

Levei isso pro Farid no Telegram e perguntei se já havia algum plano para uma base de dados hospedada pelo KDE, ou se a ideia era reaproveitar os dados do libratbag. Ele me indicou perguntar diretamente na room do KWin no Matrix.

## A discussão no Matrix

Postei a pergunta na room do KWin e recebi uma resposta detalhada do Jakob, um dos desenvolvedores do KDE:

> [Conversa no Matrix pt.1](https://matrix.to/#/!XOAbrjXKfOMhBYGSji:kde.org/$G2DNGUG3jIkKJXI57E-X2X6QHqZjxvSk5up63Q1CvM4?via=kde.org&via=matrix.org&via=im.kde.org)
> 
> [Conversa no Matrix pt.2](https://matrix.to/#/!XOAbrjXKfOMhBYGSji:kde.org/$786ZtyhCGNPUm5w9CAiw8xucH_0xBPvsEcKeqq9a0js?via=kde.org&via=matrix.org&via=im.kde.org)
> 
> [Conversa no Matrix pt.3](https://matrix.to/#/!XOAbrjXKfOMhBYGSji:kde.org/$e6MbX94HyqjZTi87sDx4Ya_6eXD4zvYRPQWxE3qI8HQ?via=kde.org&via=matrix.org&via=im.kde.org)

O resumo: não há uma resposta preferida ainda, e o espaço de design está genuinamente aberto. Algumas das tensões que ele apontou:

- **libratbag como dependência obrigatória vs. opcional**: incluir para tudo adiciona complexidade; manter opcional significa cobrir mais usuários por padrão
- **Remapeamento em software vs. firmware**: o plugin ButtonRebinds do KWin consegue remalear qualquer mouse em software, mas o remapeamento via firmware (o que o libratbag faz) persiste em qualquer sistema operacional
- **DPI vs. Pointer Speed**: os dois afetam a velocidade do cursor mas de formas fundamentalmente diferentes, e apresentar os dois na mesma UI sem confundir o usuário é um problema de UX sem solução óbvia

A conclusão dele foi basicamente: não existe resposta perfeita, tenta o melhor que conseguir e que seja uma melhoria em relação ao que existe hoje.

## Definindo a abordagem

O Farid pediu para definirmos uma proposta e postar no Invent para que um desenvolvedor desse feedback e para que ele começasse a passar a proposta para o time de design desenvolver uma interface.

[Proposta no Invent](https://invent.kde.org/teams/goals/we-care-about-your-input/-/work_items/18)

Depois de pensar nos tradeoffs, chegamos ao seguinte:

**Remapeamento de botões** será feito pelo plugin ButtonRebinds do KWin — baseado em software, funciona para qualquer mouse sem precisar do libratbag. Parecido com como jogos lidam com remapeamento de controles. A UI também vai mostrar mensagens informativas quando o usuário tentar mapear algo não suportado, como os botões esquerdo e direito do mouse.

**Ilustrações e metadados de dispositivos** virão de uma base de dados hospedada pelo KDE, espelhando o libratbag/Piper como upstream. Novos dispositivos devem ser contribuídos no libratbag primeiro. As ilustrações são desacopladas do daemon do libratbag — funcionam independentemente de ele estar instalado.

**Agrupamento de dispositivos** resolve um problema comum: mouses gaming frequentemente se registram como múltiplos dispositivos de entrada (um mouse, um teclado para os botões extras, às vezes um terceiro). O plano é agrupá-los pelo group ID do libinput para que apareçam como uma entrada única na lista. Isso requer estender a API D-Bus do KWin para expor o group ID primeiro, então é uma contribuição em dois repositórios.

**Integração com libratbag** será opcional, detectada em runtime via D-Bus. Se o daemon estiver presente, configurações adicionais ficam disponíveis: cor do LED (usando por padrão a cor de destaque que o Kameleon já define para teclados), configuração de DPI, e potencialmente remapeamento de botões via firmware para dispositivos suportados. Como o DPI deve ser apresentado ao lado do Pointer Speed existente ainda é uma questão em aberto.

## Roadmap

1. Estender a API D-Bus do KWin para expor o group ID do libinput
2. Agrupar dispositivos pelo group ID na lista do KCM
3. Montar a base de dados espelhando o libratbag/Piper
4. Integrar ilustrações de mouse usando os vendor/product IDs
5. Melhorar a UI de remapeamento: mouse→mouse, mensagens para mapeamentos não suportados
6. Adicionar integração opcional com libratbag para configurações de LED
7. Adicionar suporte a DPI via libratbag, abordagem de UI a definir

## Próximos passos

A proposta está postada e estamos aguardando feedback. Até agora tivemos uma breve olhada e pelo que parece a recepção está sendo bem positiva.



![Feedback inicial](/assets/img/posts/kde-mouse-kcm/brief-feedback.jpg)


*Leve feedback do Jakob sobre a proposta*
