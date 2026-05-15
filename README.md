# IET — Índice de Excelência Tecnológica

## Descrição

O **IET (Índice de Excelência Tecnológica)** é uma plataforma interna de avaliação contínua da maturidade tecnológica do portfólio de sistemas da companhia.

A solução foi concebida para suportar uma operação financeira regulada e de altíssima criticidade — onde indisponibilidade, falhas silenciosas em infraestrutura e divergências entre o que a arquitetura de software declara e o que a infraestrutura efetivamente entrega têm impacto direto em receita, reputação e exposição regulatória.

O objetivo central da plataforma é oferecer aos times de arquitetura, engenharia e governança uma visão consolidada, comparável e evolutiva sobre **três dimensões críticas** de cada sistema:

1. **Excelência da Arquitetura de Software** (avaliada por questionário estruturado em 6 pilares)
2. **Resiliência da Infraestrutura** (calculada a partir dos componentes que sustentam o sistema)
3. **Excelência Tecnológica Geral** — combinação ponderada das duas dimensões anteriores

Com isso, a empresa consegue:

- **Medir** o estado de maturidade de cada sistema em escala 0–100
- **Comparar** sistemas lado a lado e priorizar investimentos
- **Acompanhar evolução** ao longo do tempo (regressões e melhorias)
- **Identificar gaps recorrentes** no portfólio (perguntas onde a maioria dos sistemas falha)
- **Apoiar decisões de governança** sobre quais sistemas precisam de plano de remediação ou maior controle

---

## Arquitetura Atual

A solução é dividida em três camadas independentes, comunicando via HTTP/JSON:

```
┌───────────────────────────────────────────────────────────────────┐
│                          NAVEGADOR                                │
│                                                                   │
│   ┌─────────────────────────────────────────────────────────┐     │
│   │  iet-ui (Frontend)                                      │     │
│   │  ─────────────────────────────────────────────────      │     │
│   │  • HTML / CSS / JS puros (single page)                  │     │
│   │  • Chart.js para visualizações                          │     │
│   │  • Sem framework SPA — renderização declarativa via JS  │     │
│   │  • Hospedado em Azure Static Web Apps                   │     │
│   └────────────────────────────┬────────────────────────────┘     │
│                                │ fetch() · JSON                   │
└────────────────────────────────┼──────────────────────────────────┘
                                 │
                                 ▼
┌───────────────────────────────────────────────────────────────────┐
│  iet-backend (API)                                                │
│  ───────────────────────────────────────────────────────────      │
│  • Node.js + Express                                              │
│  • Rotas REST: /api/avaliacoes, /api/sistemas, /api/dashboard/*   │
│  • Toda a lógica de agregação está em queries SQL parametrizadas  │
│  • Cálculo dos índices vive no FRONTEND no momento do cadastro    │
│    (backend persiste os valores já calculados)                    │
│  • Hospedado em Azure Container Apps                              │
└────────────────────────────────┬──────────────────────────────────┘
                                 │ mssql (TDS)
                                 ▼
┌───────────────────────────────────────────────────────────────────┐
│  Banco de Dados                                                   │
│  ───────────────────────────────────────────────────────────      │
│  • SQL Server 2022                                                │
│  • Local (dev): container Docker (docker-compose.yaml na raiz)    │
│  • Produção: Azure SQL Database                                   │
│                                                                   │
│  Tabelas principais:                                              │
│  • sistemas       — cadastro mestre dos sistemas avaliáveis       │
│  • avaliacoes     — uma linha por avaliação realizada             │
│  • respostas      — uma linha por pergunta respondida             │
│  • componentes    — infraestrutura associada à avaliação          │
└───────────────────────────────────────────────────────────────────┘
```

### Como a solução funciona — visão geral

1. **Cadastro de sistemas**: O time mantém um inventário de sistemas críticos avaliáveis (tela "Sistemas").
2. **Cadastro de avaliação**: Para cada sistema, periodicamente (tipicamente trimestral), um arquiteto preenche o questionário (25 perguntas em 6 pilares) e cataloga os componentes de infraestrutura. Os índices são calculados em tempo real no frontend e persistidos no backend.
3. **Análise**: As demais telas consultam o backend, que agrega via SQL os resultados da última avaliação de cada sistema e expõe diferentes recortes (executivo, comparativo, por pilar, por componente, temporal).

---

## Regras de Negócio

A plataforma trabalha com **três índices principais**, todos em escala 0–100. Cada um responde a uma pergunta diferente sobre o sistema.

### Faixas de Status (aplicáveis a IET, IES, IR e scores por pilar)

| Faixa     | Range  | Significado                                                |
|-----------|--------|------------------------------------------------------------|
| Excelente | ≥ 90   | Maturidade alta, sem gaps relevantes                       |
| Bom       | 75–89  | Saudável, com pontos pontuais a melhorar                   |
| Atenção   | 60–74  | Riscos moderados — exige plano de ação                     |
| Crítico   | < 60   | Risco elevado — requer intervenção prioritária             |

---

### 1. IES — Índice de Excelência de Software

**Pergunta que responde:** *"Quão madura é a arquitetura de software deste sistema?"*

**Como é calculado:**

O IES nasce do questionário de 25 perguntas estruturado em **6 pilares**, cada um com peso fixo:

| Pilar | Nome completo                          | Peso  | Nº perguntas |
|-------|----------------------------------------|-------|--------------|
| P1    | Topologia & Ponto Único de Falha       | 30%   | 4            |
| P2    | Dados Distribuídos                     | 25%   | 4            |
| P3    | Operabilidade                          | 15%   | 4            |
| P4    | Arquitetura Evolutiva                  | 10%   | 5            |
| P5    | Observabilidade                        | 10%   | 4            |
| P6    | Segurança                              | 10%   | 4            |

Os pesos refletem a importância relativa de cada dimensão para a continuidade do negócio. Topologia tem o maior peso porque é a primeira linha de defesa contra indisponibilidade; Segurança e Observabilidade, embora críticas, têm peso menor pois geralmente possuem mecanismos compensatórios (controles externos, frameworks corporativos).

**Cálculo passo a passo (com exemplo):**

1. **Score por pergunta**: cada pergunta tem múltiplas opções (A–E) com score associado (0 a 10). O respondente pode marcar uma ou mais opções; o score da pergunta é a **média aritmética** das opções selecionadas.
   ```
   score_pergunta = média(scores_opções_selecionadas)   // escala 0–10
   ```
   *Exemplo:* na pergunta `P1Q1_TOPO`, o arquiteto marca as opções **C (score 5)** e **D (score 8)**.
   `score_pergunta = (5 + 8) / 2 = 6.5`

2. **Score por pilar**: dentro de um pilar, faz-se a média aritmética das perguntas respondidas, multiplicada por 10 para normalizar à escala 0–100.
   ```
   score_pilar = média(scores_perguntas_do_pilar) × 10  // escala 0–100
   ```
   *Exemplo:* o pilar **P1 Topologia** tem 4 perguntas. As respostas resultaram nos scores `6.5`, `8.0`, `5.0` e `7.5`.
   `score_pilar_P1 = ((6.5 + 8.0 + 5.0 + 7.5) / 4) × 10 = 6.75 × 10 = 67.5`

3. **IES**: soma **ponderada** dos 6 scores de pilar, usando os pesos acima. Como Σ pesos = 1.0, a soma já é o IES final.
   ```
   IES = Σ (score_pilar_i × peso_pilar_i)        // pilar não respondido contribui com 0
   ```
   *Exemplo:* o portfólio termina com os seguintes scores por pilar:

   | Pilar | Score | Peso  | Contribuição    |
   |-------|-------|-------|-----------------|
   | P1    | 67.5  | 0.30  | 20.25           |
   | P2    | 80.0  | 0.25  | 20.00           |
   | P3    | 70.0  | 0.15  | 10.50           |
   | P4    | 60.0  | 0.10  |  6.00           |
   | P5    | 75.0  | 0.10  |  7.50           |
   | P6    | 50.0  | 0.10  |  5.00           |
   | **Σ** |       | **1.00** | **69.25**    |

   `IES = 69.25` → faixa **Atenção** (60–74).

   **Pilares não respondidos contam como zero** — não há renormalização. Isso é proposital: deixar de avaliar uma dimensão é tão informativo quanto avaliá-la mal, e a nota precisa refletir essa lacuna. Por exemplo, se P6 (Segurança, peso 0.10) não tivesse sido respondido no exemplo acima, o cálculo seria `20.25 + 20 + 10.5 + 6 + 7.5 + (0 × 0.10) = 64.25`. A nota cai 5 pontos pelo pilar omitido — exatamente o peso de P6.

**Como interpretar:**

- IES isolado **não** captura riscos de infraestrutura — um sistema pode ter arquitetura "bonita no papel" e infra precária. Por isso o IET combina IES com IR.
- A decomposição por pilar (tela "Análise por Pilar") é onde se identificam **gaps sistêmicos** do portfólio. Se Segurança tem nota média 45 no portfólio, há um problema horizontal a ser tratado.

---

### 2. IR — Índice de Resiliência

**Pergunta que responde:** *"Quão resistente a falhas é a infraestrutura que sustenta este sistema?"*

**De onde vem:**

Diferente do IES (que vem de um questionário), o IR é **calculado a partir dos componentes de infraestrutura cadastrados** durante a avaliação. Cada componente recebe atributos qualitativos (redundância, tolerância a falhas, backup/RPO, DR) e a plataforma deriva scores numéricos automaticamente.

**Atributos cadastrados por componente:**

| Atributo          | Origem                                                   |
|-------------------|----------------------------------------------------------|
| Nome              | Texto livre (ex: "Oracle Database 19c", "Redis Cluster") |
| Tipo              | IaaS, PaaS, SaaS, FaaS, DBaaS                           |
| Provedor          | Azure, AWS, On-Premise                                  |
| Impacto           | Baixo, Médio, Alto, Extremo (escala 1–4)                |
| Redundância       | escala qualitativa de 5 níveis                          |
| Tolerância falhas | escala qualitativa de 5 níveis                          |
| Backup/RPO        | escala qualitativa de 5 níveis                          |
| DR (Recuperação)  | escala qualitativa de 5 níveis                          |
| Caminho Crítico   | Core (sistema central) ou Satélite                      |
| Happy Socks       | classificação Verde/Amarelo/Vermelho de criticidade     |

**Cálculos derivados por componente:**

```
P (Score Resiliência 1–5) = média(redundância, tolerância, backup, DR) + 1
Q (Score 0–100)           = P × 20                  // normalização para 0–100
R (Score Risco 0–100)     = (impacto × happy_socks) / 12 × 100
S (Classificação Risco)   = Verde (R<30) | Amarelo (R<65) | Vermelho (R≥65)
T (Peso Impacto)          = 0.5 | 1.0 | 1.5 | 2.0   // por nível de impacto
U (Peso Criticidade)      = depende se Core (1.5–3.0) ou Satélite (0.8–1.5) × impacto
V (Peso Tipo Solução)     = 1.5 (Core) | 0.5 (Satélite)
```

**Cálculo do IR (média ponderada dos componentes):**

```
IR = Σ (score_100_componente × peso_crit_componente) / Σ peso_crit_componente
```

A ponderação por **peso de criticidade (U)** é o que faz a fórmula ser justa: um banco de dados Core com Impacto Extremo pesa mais (até 3.0) do que um cache Satélite Baixo Impacto (0.8). Assim, um componente Core mal configurado afunda o IR proporcionalmente ao seu impacto real no sistema.

**Score de Risco (R) — uso secundário:**

Embora IR meça resiliência *positiva* (o quanto está bom), o **Score de Risco** (R) faz o oposto: combina impacto de negócio com a saúde operacional declarada (Happy Socks). É usado na tela "Componentes & Riscos" para destacar componentes que pedem ação imediata, com 5 faixas:

| Faixa        | Range  |
|--------------|--------|
| Muito Baixo  | < 20   |
| Baixo        | 20–39  |
| Médio        | 40–59  |
| Alto         | 60–79  |
| Muito Alto   | ≥ 80   |

---

### 3. IET — Índice de Excelência Tecnológica

**Pergunta que responde:** *"Considerando arquitetura E infraestrutura, quão maduro tecnologicamente é este sistema?"*

**Fórmula:**

```
IET = IES × 0.6 + IR × 0.4
```

A ponderação **60/40 favorece IES** porque a maturidade arquitetural costuma ser preditora mais estável da saúde de longo prazo do sistema — infraestrutura pode ser corrigida pontualmente, mas vícios de arquitetura tendem a se propagar e exigir reescritas. Os 40% do IR garantem que sistemas com arquitetura excelente mas infra precária não sejam "premiados" injustamente.

O IET é a métrica primária exibida em quase todas as visões da plataforma e é o que classifica o sistema na faixa de status (Excelente / Bom / Atenção / Crítico).

---

### Como funcionam as perguntas

- **Estrutura**: cada pergunta tem um identificador único (ex: `P1Q1_TOPO`), pertence a um pilar, e oferece de 4 a 5 alternativas (A, B, C, D, E) com **score crescente** (alternativa A geralmente vale 0 ou 2; alternativa E vale 10 — cenário ideal).
- **Múltipla escolha**: o respondente pode marcar mais de uma alternativa quando o cenário real combina características — o score da pergunta vira a média das selecionadas. Isso permite respostas honestas em sistemas heterogêneos.
- **Persistência**: na tabela `respostas`, ficam armazenados o `questao_id`, o array JSON das letras selecionadas e o score numérico. Os textos das opções também são gravados (`opcoes_texto`) para que a avaliação seja auto-contida — mesmo que o questionário evolua, é possível reler a avaliação histórica com os textos da época.
- **Versionamento das perguntas**: o catálogo de perguntas vive no frontend (`QUESTIONS` em [index.html](index.html)). Mudanças de redação não afetam avaliações já gravadas; mudanças de score/peso afetam apenas avaliações **futuras**.

---

### Por que cadastrar componentes de infraestrutura

O cadastro de componentes não é só para calcular o IR. Ele permite responder perguntas executivas que o questionário sozinho não responde:

- **"Quais componentes do portfólio são pontos cegos de risco?"** — tela Componentes & Riscos, ordenada por Score de Risco
- **"Onde estamos concentrando nuvem?"** — distribuição por provedor e tipo de serviço
- **"Quais sistemas têm mais dependência de provedor único?"** — cruzando provedor × sistema
