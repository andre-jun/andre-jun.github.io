---
title: "Análise exploratoria de dados da LKML5Ws"
date: 2026-06-25 19:21:33 -0300
categories: [Kernel, Análise]
tags: [python, nlp, vader, sentiment, amd-gfx, kernel, polars, dataset]
---

Como parte da disciplina MAC0470, fizemos uma análise de sentimentos sobre
emails da mailing list `amd-gfx` usando o dataset
[LKML5Ws](https://files.rcpassos.me/public/Academic/Datasets/LKML5Ws_uncompressed_lists/). O objetivo foi entender como
o tom das mensagens varia entre threads, remetentes e ao longo do tempo numa
lista de desenvolvimento de kernel.

O trabalho foi feito em conjunto com Gabriel Dimant e Guilherme Gabriel.

## O dataset

O LKML5Ws agrega emails de várias mailing lists do kernel Linux em formato
Parquet, com colunas como `message_id`, `in_reply_to`, `from`, `subject`,
`date`, `raw_body` e `has_patch_tag`. Filtramos apenas os emails com
`has_patch_tag = True` para facilitar a análise — patches são melhor
formatados que discussões livres e permitem comparações mais consistentes.

## Reconstruindo as threads

O dataset armazena emails individualmente, sem agrupamento por thread. Para
reconstruir as threads, usamos a coluna `in_reply_to`, que em respostas aponta
para o `message_id` da mensagem pai. A partir disso, adicionamos uma coluna
`thread_id` com o `message_id` raiz de cada conversa:

```python
def add_thread_ids(df: pl.DataFrame) -> pl.DataFrame:
    parent = dict(zip(df["message_id"].to_list(), df["in_reply_to"].to_list()))
    cache = {}

    def find_root(msg_id):
        if msg_id is None:
            return None
        if msg_id in cache:
            return cache[msg_id]
        current = msg_id
        visited = set()
        while True:
            if current not in parent:
                root = current; break
            p = parent[current]
            if p is None:
                root = current; break
            if p not in parent:
                root = p; break
            if current in visited:
                root = current; break
            visited.add(current)
            current = p
        for node in visited:
            cache[node] = root
        cache[msg_id] = root
        return root

    return df.with_columns(pl.Series("thread_id", [find_root(m) for m in df["message_id"]]))
```

## Limpando o corpo dos emails

A coluna `raw_body` contém o email inteiro: texto citado de respostas
anteriores, trailers (`Signed-off-by`, `Reviewed-by`, etc.), diffs, cabeçalhos
automáticos da mailing list e a mensagem propriamente dita. Tudo isso interfere
na análise de sentimento, então escrevemos uma função de limpeza que remove
esses elementos linha a linha:

```python
DIFF_MARKERS = ("diff --git", "Index:", "@@")

PATTERN = re.compile(
    r"^(Signed[- ]off[- ]by|Acked[- ]by|Reviewed[- ]by|"
    r"Suggested[- ]by|Tested[- ]by|Reported[- ]by|"
    r"Fixes|Link|Co[- ]developed[- ]by):"
)

def clean_email(body: str) -> str:
    if body is None:
        return ""
    cleaned = []
    for line in body.splitlines():
        if (line.lstrip().startswith(">")
                or line.startswith("From:")
                or line.startswith("[AMD Official Use Only - General]")
                or line.startswith(("On Mon", "On Tue", "On Wed",
                                    "On Thu", "On Fri", "On Sat", "On Sun"))):
            continue
        if (PATTERN.match(line) or line.strip() == "--"
                or line.startswith("-----Original Message-----")
                or line.startswith("Sent:")):
            break
        if line == "---" or any(line.startswith(m) for m in DIFF_MARKERS):
            break
        cleaned.append(line)
    text = "\n".join(cleaned)
    text = re.sub(r"\n{3,}", "\n\n", text)
    text = re.sub(r"[ \t]+", " ", text)
    return text.strip()
```

A limpeza ainda é um trabalho em progresso — alguns emails ainda escapam com
conteúdo indesejado, especialmente quando a formatação foge do padrão esperado.

## Scoring com VADER

A ferramenta escolhida foi o
[VADER](https://github.com/cjhutto/vaderSentiment), um analisador de
sentimentos baseado em regras e léxico, sem necessidade de GPU ou modelos
pesados. Ele retorna um score composto (`compound`) no intervalo [-1, 1].

Tentamos também o RoBERTa (`cardiffnlp/twitter-roberta-base-sentiment-latest`),
que é mais robusto, mas o tempo de inferência sobre ~134k mensagens foi
proibitivo no hardware disponível. Seguimos com o VADER.

```python
from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer

vader = SentimentIntensityAnalyzer()

def vader_score(text: str) -> float:
    if not text or text.isspace():
        return 0.0
    return vader.polarity_scores(text)["compound"]
```

O dataset processado e pontuado foi salvo em Parquet para a etapa de análise.

## Resultados

### Estatísticas gerais

Com o dataset completo (134.774 mensagens), o score médio foi levemente
positivo:

| Métrica | Valor  |
|---------|--------|
| Média   | 0.1213 |
| Mediana | 0.0    |
| Desvio  | 0.4443 |
| Mínimo  | -1.0   |
| Máximo  | 1.0    |

Usando o threshold recomendado pela documentação do VADER (±0.05):

| Classe   | Mensagens |
|----------|-----------|
| Positive | 59.420    |
| Neutral  | 44.268    |
| Negative | 31.086    |

### Threads mais negativas e mais positivas

Agrupando por `thread_id` e calculando a média de sentimento por thread, os
extremos foram inspecionados manualmente. As threads mais negativas e mais
positivas tinham todas apenas 1 mensagem — o que já levantou suspeitas sobre
a qualidade desses resultados como representativos da lista.

### O problema dos outliers

Ao inspecionar os casos mais extremos (scores próximos de -1 ou +1), ficou
claro que a maioria é ruído: patches com código ou terminologia muito
específica que o VADER interpreta como emocionalmente carregada, sem que haja
de fato um tom negativo ou positivo na mensagem.

Isso faz sentido: o VADER foi desenvolvido para textos informais de redes
sociais, muito mais expressivos emocionalmente do que emails técnicos de kernel.

Para isolar os casos mais informativos, filtramos as threads com média de
sentimento fora do intervalo [-0.5, 0.5]:

```
Threads totais   : 39.445
Threads outliers : 7.370  (18,7%)
Threads normais  : 32.075 (81,3%)
```

Repetindo as análises apenas nas threads "normais", os resultados ficaram mais
coerentes com o que se esperaria da lista.

### Remetentes mais negativos e mais positivos

Filtrando apenas remetentes com ao menos 10 mensagens, os rankings de
sentimento médio revelaram alguns padrões interessantes — e alguns casos
claramente afetados pelo mesmo problema de outliers. Entre os "mais negativos"
apareceram inclusive contribuidores brasileiros da USP, o que sugere que o
conteúdo dos emails (patches com muito código ou texto técnico denso) ainda
está influenciando o score mais do que o tom em si.

Mais positivos - sem outliers:

| from | messages | threads | mean_sentiment | std |
|------|---------:|--------:|---------------:|----:|
| Cihangir Akturk <cakturk-Re5JQEeQqe8AvxtiuMwx3w@public.gmane.org> | 2 | 2 | 0.99785 | 0.000071 |
| Andreas Messer <andi@bastelmap.de> | 1 | 1 | 0.976 | 0.0 |
| Dan Carpenter via dri-devel <dri-devel@lists.freedesktop.org> | 1 | 1 | 0.9732 | 0.0 |
| Amit Kachhap <Amit.Kachhap@arm.com> | 1 | 1 | 0.9699 | 0.0 |
| Mario Kleiner via dri-devel <dri-devel@lists.freedesktop.org> | 1 | 1 | 0.9549 | 0.0 |
| Daniel Latypov <dlatypov@google.com> | 1 | 1 | 0.9546 | 0.0 |
| ydirson@free.fr | 1 | 1 | 0.9506 | 0.0 |
| Sebin Sebastian <mailmesebin00@gmail.com> | 1 | 1 | 0.9477 | 0.0 |
| patchwork-bot+netdevbpf@kernel.org | 1 | 1 | 0.9422 | 0.0 |
| liviu.dudau@arm.com <liviu.dudau@arm.com> | 1 | 1 | 0.9412 | 0.0 |
| Jeff Cook <jeff@jeffcook.io> | 1 | 1 | 0.9374 | 0.0 |
| Andrew Worsley <amworsley@gmail.com> | 1 | 1 | 0.9311 | 0.0 |
| taskboxtester@gmail.com | 1 | 1 | 0.9265 | 0.0 |
| Vivek Das Mohapatra <vivek@collabora.com> | 2 | 1 | 0.9186 | 0.0 |
| Welty Brian <brian.welty@intel.com> | 1 | 1 | 0.9095 | 0.0 |
| Akash Goel <akash.goel@arm.com> | 3 | 1 | 0.908567 | 0.056407 |
| Liu HaoPing (Alan) <HaoPing.liu@amd.com> | 1 | 1 | 0.9062 | 0.0 |
| Tamminen Eero T <eero.t.tamminen@intel.com> | 1 | 1 | 0.9042 | 0.0 |
| Linus Torvalds <torvalds-de/tnXTf+JLsfHDXvbKv3WD2FQJk+8+b@public.gmane.org> | 1 | 1 | 0.9032 | 0.0 |
| Zhang Tiantian (Celine) <Tiantian.Zhang@amd.com> | 1 | 1 | 0.9022 | 0.0 |
    
Mais negativos - sem outliers:

| from | messages | threads | mean_sentiment | std |
|------|---------:|--------:|---------------:|----:|
| kbuild test robot via dri-devel <dri-devel@lists.freedesktop.org> | 1 | 1 | -0.9999 | 0.0 |
| Michael D Labriola <michael.d.labriola@gmail.com> | 1 | 1 | -0.9762 | 0.0 |
| ozeng <ozeng-5C7GfCeVMHo@public.gmane.org> | 1 | 1 | -0.9739 | 0.0 |
| Michael Bommarito <michael.bommarito@gmail.com> | 1 | 1 | -0.9732 | 0.0 |
| Jérôme Glisse <jglisse-H+wXaHxf7aLQT0dZR+AlfA@public.gmane.org> | 1 | 1 | -0.9675 | 0.0 |
| Liu01 Tong <Tong.Liu01@amd.com> | 3 | 3 | -0.9644 | 0.021997 |
| David Baum <davidbaum461@gmail.com> | 1 | 1 | -0.9565 | 0.0 |
| Amir Shetaia <amir.shetaia@amd.com> | 1 | 1 | -0.9509 | 0.0 |
| Mario Limonciello <superm1@gmail.com> | 1 | 1 | -0.9501 | 0.0 |
| xurui <xurui@kylinos.cn> | 2 | 2 | -0.9452 | 0.0 |
| R SUNDAR <prosunofficial@gmail.com> | 1 | 1 | -0.9413 | 0.0 |
| Swarup Laxman Kotiaklapudi <swarupkotikalapudi@gmail.com> | 1 | 1 | -0.9413 | 0.0 |
| ts8060 <ts8060@my.bristol.ac.uk> | 1 | 1 | -0.9386 | 0.0 |
| Dixit Ashutosh <ashutosh.dixit@intel.com> | 1 | 1 | -0.9349 | 0.0 |
| Guangshuo Li <lgs201920130244@gmail.com> | 2 | 2 | -0.92785 | 0.045467 |
| Siyang Liu <Security@tencent.com> | 1 | 1 | -0.9274 | 0.0 |
| Ahmed Elmetwally <en22ue@gmail.com> | 1 | 1 | -0.9246 | 0.0 |
| Ken Xue <Ken.Xue@amd.com> | 1 | 1 | -0.9062 | 0.0 |
| Ziyi Guo <n7l8m4@u.northwestern.edu> | 1 | 1 | -0.9055 | 0.0 |
| Daniel Kurtz <djkurtz-F7+t8E8rja9g9hUCZPvPmw@public.gmane.org> | 1 | 1 | -0.8989 | 0.0 |

## Limitações e próximos passos

- A limpeza do `raw_body` ainda não é perfeita e afeta a qualidade do scoring
- O VADER não foi treinado em linguagem técnica de kernel — scores extremos
  devem ser interpretados com cautela
- Uma direção interessante seria usar embeddings ou modelos fine-tuned em
  texto técnico para comparar com os resultados do VADER
- A análise de evolução temporal do sentimento por thread ainda não foi feita
