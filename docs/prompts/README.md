# Os prompts da live

A sequência que constrói o Data Intake Validator, na ordem em que será usada. Cada prompt é
autocontido: copie o arquivo inteiro, cole no Claude Code dentro do repositório, e deixe
rodar.

**Não pule a ordem.** Cada prompt assume que o anterior terminou — o 02 lê o contrato que o
01 escreveu, o 05 grava no catálogo que o 00 criou.

---

## O mapa

| # | Prompt | O que produz | Onde toca | Tempo | Estado |
| --- | --- | --- | --- | --- | --- |
| 00 | [setup](00-setup.md) | catálogo, 4 schemas, Volume `entrada` | workspace Databricks | 8 min | escrito |
| 01 | [contrato](01-contrato.md) | `docs/contrato-de-dados.md` | repositório | 10 min | escrito |
| 02 | [validação](02-validacao.md) | `validator/` e o verificador | repositório | 12 min | escrito |
| 03 | [app](03-app.md) | o App Streamlit implantado como Databricks App | `apps/` + workspace | 12 min | escrito |
| 04 | [ponte](04-ponte.md) | o `.xlsx` aprovado virando `.csv` no Volume | `ingestion/` + Volume | 6 min | escrito |
| 05 | [bronze](05-bronze.md) | `bronze.vendas` com metadados técnicos | `notebooks/` + warehouse | 8 min | escrito |
| 06 | [silver](06-silver.md) | `silver.vendas` tipada | `notebooks/` + warehouse | 8 min | escrito |
| 07 | [gold](07-gold.md) | `vendas_por_produto` e a consulta final | `notebooks/` + warehouse | 8 min | escrito |

Os prompts somam ~72 minutos; com a cena das decisões (~8 min, que abre a live), são ~80.
Os tempos e o roteiro da apresentação ficam fora deste repositório.

**O prompt 00 é o único que pede confirmação antes de executar** — ele cria o catálogo, e
criar é barato; desfazer é que não é. Ele também é o único que não escreve nada no
repositório: os prompts 03 a 07 escrevem no repositório *e* alteram a workspace (o App, o
Volume, as tabelas). As camadas 05–07 rodam SQL pelo `scripts/executar_sql.py`, não por Jobs
nem por Pipelines — a decisão 6.13 explica o porquê.

---

## O ato que não tem prompt

As decisões não estão em nenhum arquivo desta pasta. Elas acontecem **antes do prompt 00**:
você abre o `projeto.md`, pede ao Claude para levantar os pontos em aberto **sem consultar o
`decisoes.md`**, decide três ou quatro ao vivo, e só então consulta a referência construída.

**Por que antes do 00.** A cena só funciona enquanto o Claude não tiver lido a referência construída. O
prompt 00 aponta para as decisões 6.7 e 6.8, e o `CLAUDE.md` também — rodando depois dele, é
provável que a sessão já tenha aberto o `decisoes.md`, e aí a cena morre. É também o motivo
de a sessão da live ter de ser **nova**: uma que já trabalhou no projeto já leu o arquivo.

O texto exato do prompt a usar fica fora deste repositório. É a tese do projeto — a IA levanta as ambiguidades, o humano decide — e o
único momento da live em que ela aparece acontecendo.

---

## Antes da live: o ponto de partida

**Decidido: o contrato e o módulo de validação nascem ao vivo.** A live reconstrói a
sequência inteira, 00 a 07 — não há atalho pelo 03.

O que isso exige do repositório antes de começar:

| Onde | O que tem |
| --- | --- |
| `main` (ponto de partida) | especificação, decisões, gerador, os 5 `.xlsx`, estes prompts e os scripts. **Sem** `docs/contrato-de-dados.md` e sem `validator/` |
| versão construída | fica fora deste repositório, para comparação |

O ciclo de trabalho é: resetar o repositório para o ponto de partida → criar a branch do
ensaio → rodar os prompts → comparar com a referência → descartar o ensaio. O procedimento
exato fica fora deste repositório, e o checklist que
verifica se o repositório está no ponto certo é
`python scripts/verificar_estado_live.py --profile <perfil>` — ele confere que os artefatos
nascidos-ao-vivo estão ausentes antes de cada ensaio e no dia da live.

**Se você colar o prompt 01 com o `docs/contrato-de-dados.md` já presente, o Claude vai
encontrar um arquivo que o prompt manda criar.** É exatamente o que o checklist pega.
