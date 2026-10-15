# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Idioma

O projeto é escrito em português (pt-BR): documentação, comentários, nomes de tabelas e colunas, mensagens de interface. Mantenha essa convenção em código e documentação novos.

## O que é este projeto

**Data Intake Validator** — solução demonstrativa de Engenharia de Dados para uma live. Simula o cenário de empresas que recebem planilhas Excel de diferentes áreas e precisam validá-las contra um contrato de dados antes de permitir a ingestão.

Fluxo: `Usuário → Validação → Ingestão → Bronze → Silver → Gold → Consumo`

O contrato completo (objetivo, problema de negócio, schema, escopo, critérios de conclusão) está em [docs/projeto.md](docs/projeto.md). **É a fonte de verdade sobre o que deve existir — leia antes de implementar qualquer coisa.**

As decisões tomadas sobre os pontos em aberto do contrato estão em [docs/decisoes.md](docs/decisoes.md) — **as vinte e duas estão fechadas**. **Onde os dois divergirem, `decisoes.md` prevalece** — foi ele que resolveu as ambiguidades do texto original (por exemplo, `desconto` é fração 0–1, não percentual).

## Estado atual

> Esta seção descreve o ponto de partida. A versão construída do projeto fica fora deste repositório.

**Este repositório é o ponto de partida da live.** O contrato de dados e o módulo de validação **nascem ao vivo**, nos prompts 01 e 02 — não existem aqui, e é assim que deve ser. Não recrie esses arquivos no `main`: a ausência deles é o que o checklist de partida verifica.

Existem a especificação, o registro de decisões, o gerador do dataset, os cinco arquivos de teste, os prompts 00 a 07 e os scripts da live.

Os diretórios `validator/`, `apps/`, `ingestion/` e `notebooks/` estão vazios e representam o layout-alvo — não são sobras nem placeholders a remover. O `validator/` se enche no prompt 02; `apps/` e `ingestion/`, nos prompts 03 e 04; `notebooks/` recebe os três `.sql` das camadas, nos prompts 05 a 07.

Ainda não há `pyproject.toml` nem `databricks.yml`. Na workspace, o catálogo `lakehouse_validator` também ainda não existe.

**Ordem da live:** a **cena das decisões** abre a live — antes do prompt 00, numa sessão nova, enquanto o Claude ainda não leu `docs/decisoes.md`; depois 00 (cria o catálogo **pela UI** — a conta usa Default Storage e a plataforma não deixa a CLI criar catálogos assim — e depois os 4 schemas e o Volume pela CLI; é o único passo que pede confirmação antes de executar), 01 e 02 (contrato e validação, só no repositório), 03 (App), 04 (ponte) e 05 a 07 (as camadas, via `scripts/executar_sql.py`).

## Sequência de prompts

Os prompts que constroem o projeto, na ordem em que serão usados na live, ficam em `docs/prompts/` — o índice com o mapa da sequência, os tempos e o estado de cada um é [docs/prompts/README.md](docs/prompts/README.md).

## Comandos

### Setup

Ambiente: Python 3.14.5, pandas 3.0.6, numpy 2.5.3, openpyxl 3.1.5 — já instalados. As versões estão fixadas em [requirements.txt](requirements.txt), e não é por formalidade: a geração dos `.xlsx` depende de comportamento específico do pandas 3.

```bash
python -m pip install -r requirements.txt
```

### Gerar o dataset e os arquivos de teste

```bash
python scripts/generate_sales.py
```

`SEED = 42` e `N_RECORDS = 1000` em [scripts/generate_sales.py](scripts/generate_sales.py) — o dataset é reproduzível, e alterar `N_RECORDS` muda o volume sem mudar o modelo. Os arquivos gerados não devem ser editados à mão.

`OUTPUT_DIR` é ancorado na raiz do projeto via `__file__`, então o script pode ser rodado de qualquer diretório e sempre grava em `data/output/` — nunca em uma pasta `output/` no diretório de execução.

Três armadilhas já pagas neste projeto:

- **O gerador exige `astype(object)` explícito** antes de injetar texto em coluna numérica. O pandas 3 removeu o upcast automático e a atribuição levanta `TypeError` — o script simplesmente não rodava neste ambiente. O upcast é o defeito que `vendas_tipo_invalido.xlsx` precisa carregar, então ele é explícito no código, e não efeito colateral da versão da biblioteca.
- **Leia os `.xlsx` com `keep_default_na=False, na_filter=False, dtype=object`.** Sem `keep_default_na`, o pandas converte `"N/A"` — o texto gravado na linha 514 de `vendas_tipo_invalido.xlsx` (índice 512 do DataFrame, mais o cabeçalho) — em `nan`, e o diagnóstico relata "campo vazio" onde o arquivo diz `"N/A"`. Sem `dtype=object`, entram os tipos inferidos: `np.int64`, `Timestamp`.
- **`isinstance(np.int64(1), int)` é `False`.** É a consequência prática da armadilha anterior, e a mais perigosa das três: um validador que confie em `isinstance` sobre valores lidos sem `dtype=object` reporta **todo inteiro do arquivo** como tipo inválido. Não quebra nenhum arquivo pequeno; quebra ao vivo, nos dois arquivos que deveriam ser aprovados. A receita completa está em [docs/decisoes.md](docs/decisoes.md) §6.21.

Os hashes dos `.xlsx` **não** são estáveis entre versões de pandas e openpyxl — os mesmos dados gravados por outra versão produzem bytes diferentes. Compare conteúdo, nunca hash, ao verificar se o gerador mudou.

### Checklist de partida da live

```bash
python scripts/verificar_estado_live.py --profile <perfil>
```

Confere o repositório no ponto de partida (branch `main`, sem alterações pendentes, artefatos que nascem ao vivo ausentes, os 5 `.xlsx` presentes) e o ambiente (pandas, CLI autenticado, catálogo `lakehouse_validator` ainda não criado, warehouse visível). Rode **antes de cada ensaio e no dia da live**; sai com código 1 se algo estiver fora do lugar.

O verificador do módulo de validação (`scripts/verificar_validator.py`, 22 casos) nasce no prompt 02 — ele não existe neste ponto de partida.

### Rodar SQL no warehouse

```bash
python scripts/executar_sql.py notebooks/01_bronze.sql --profile <perfil>
python scripts/executar_sql.py --sql "SELECT COUNT(*) FROM lakehouse_validator.bronze.vendas" --profile <perfil>
```

Executa **um** statement por chamada — o arquivo `.sql` não é dividido por `;`. Sobe o warehouse sozinho se estiver parado (serverless) e imprime o resultado alinhado em colunas. A substring `__ID_EXECUCAO__`, quando presente no SQL, vira um id único da execução (decisão 6.22). É assim que as camadas rodam na live: sem Jobs e sem Pipelines — a decisão 6.13 explica o porquê.

### Arquivos de teste (em `data/output/`)

| Arquivo | Defeito injetado | Veredito | O que exercita |
| --- | --- | --- | --- |
| `vendas_valido.xlsx` | nenhum | aprovado | caminho feliz → `READY FOR INGESTION` |
| `vendas_data_texto.xlsx` | nenhum — só `data_venda` como texto `yyyy-MM-dd` | aprovado | a segunda forma de data do contrato (§4.3); prova que o contrato aceita duas formas, não que ele checa formato |
| `vendas_coluna_ausente.xlsx` | `data_venda` renomeada para `dt_venda` | rejeitado | coluna obrigatória ausente + coluna inesperada; é o caso que pede sugestão de correção |
| `vendas_tipo_invalido.xlsx` | linhas 19/145/514 com `quantidade` textual (`"três"`, `"dez"`, `"N/A"`); linhas 90/378 com `valor_unitario` textual (`"1.299,90"`, `"valor_invalido"`) | rejeitado | validação de tipo em 5 linhas de 1000 — o arquivo inteiro é rejeitado |
| `vendas_coluna_extra.xlsx` | coluna adicional `observacao_interna` | rejeitado | coluna fora do contrato |

Todos derivam do mesmo dataset base, para que as diferenças entre eles representem exclusivamente o defeito demonstrado. Preserve essa propriedade ao alterar o gerador.

Dois são aprovados e três rejeitados, cobrindo **três motivos distintos** de rejeição: coluna obrigatória ausente (com sugestão de correção), coluna inesperada e tipo inválido. É isso que dá variedade à demonstração.

## Contrato de dados

13 colunas. **Antes do prompt 01, a autoridade sobre o schema é [docs/projeto.md](docs/projeto.md) §3 e, sobre as ambiguidades, [docs/decisoes.md](docs/decisoes.md)** — as 22 decisões, que prevalecem onde os dois divergirem. A partir do prompt 01, o contrato legível passa a ser `docs/contrato-de-dados.md`; a partir do 02, a versão em código é `validator/contrato.py`, e **é o único lugar onde uma regra pode ser declarada** — nenhuma camada redefine o que está lá.

| Campo | Tipo |
| --- | --- |
| `id_venda` | integer |
| `data_venda` | date |
| `id_cliente` | integer |
| `cliente` | string |
| `cidade` | string |
| `estado` | string |
| `produto` | string |
| `categoria` | string |
| `quantidade` | integer |
| `valor_unitario` | decimal |
| `desconto` | decimal |
| `vendedor` | string |
| `canal_venda` | string |

## Arquitetura

### Separação app ↔ validação (requisito central)

A lógica de validação **não pode depender do Streamlit**. Ela deve ser um módulo independente, importável por qualquer mecanismo de ingestão, e o App apenas consome o resultado:

```
App (Streamlit) → validação (Python puro) → resultado → ingestão
```

Ao adicionar uma regra de validação, ela vai no módulo de validação — o App só renderiza. É isso que permite reusar a validação fora do App.

O módulo — que nasce no prompt 02 — terá três arquivos, e a divisão é a decisão 6.4 tornada visível:

| Arquivo | Papel |
| --- | --- |
| `validator/contrato.py` | só **declara** — as 13 colunas, os tipos, os domínios, as faixas, as nove regras. Zero lógica |
| `validator/validador.py` | só **aplica** — lê o arquivo, percorre as células, monta o `Diagnostico` |
| `validator/__init__.py` | a única porta de entrada. O App importa daqui |

Quem consome não precisa saber que são dois arquivos. E quem for explicar na live pode mostrar a separação: um arquivo que se lê como o contrato, outro que se lê como o motor.

### Responsabilidades das camadas

| Camada | Faz | Não faz |
| --- | --- | --- |
| App | upload de `.xlsx`, ler arquivo, exibir diagnóstico, aprovar/rejeitar | conter regra de validação |
| Validação | colunas obrigatórias, colunas inesperadas, tipos, datas, valores numéricos | corrigir dados |
| Ingestão | deixar entrar apenas arquivo aprovado | aceitar arquivo rejeitado |
| Bronze | preserva o dado recebido + metadados técnicos (timestamp de ingestão, arquivo de origem, id da execução) | aplicar regra de negócio |
| Silver | tipos corretos, padronização, tratamento de nulos, limpeza de strings, qualidade técnica | inventar regra de negócio |
| Gold | agregações orientadas a consumo | criar tabelas sem necessidade |

Tabelas: `lakehouse_validator.bronze.vendas`, `lakehouse_validator.silver.vendas`, `lakehouse_validator.gold.vendas_por_produto` (produto, categoria, quantidade de vendas, quantidade total vendida, faturamento total, desconto médio, ticket médio). O catálogo é dedicado a este projeto — ver [docs/decisoes.md](docs/decisoes.md) §6.8.

### Comportamento obrigatório da validação

- Aprovado → `READY FOR INGESTION`
- Rejeitado → `FILE REJECTED`
- **Nunca corrigir dados silenciosamente.** Sugestões são permitidas desde que explícitas e sujeitas à decisão do usuário — ex.: "`data_venda` ausente; possível correspondência `dt_venda`; renomear `dt_venda` → `data_venda`".

## Restrições do projeto

Derivadas de [docs/projeto.md](docs/projeto.md) §12–13. São requisitos, não preferências de estilo:

- **Não inventar regras de negócio.** Transformações precisam ser justificadas pelo contrato. A IA não é autoridade sobre schema, contrato, regras de validação, responsabilidades das camadas ou critérios de qualidade.
- **Não adicionar** autenticação, APIs externas, notificações, histórico de uploads, versionamento de schema, auto-correção, múltiplos formatos de arquivo nem infraestrutura desnecessária. Isso é escopo futuro (§15) e não deve aumentar a complexidade da primeira versão sem necessidade.
- **Simplicidade e explicabilidade acima de tudo.** O projeto existe para ser explicado numa live — evite abstrações excessivas e automações que dificultem a explicação.
- **Não criar múltiplas tabelas Gold sem necessidade.**

## Databricks

CLI do Databricks. Os perfis ficam no `.databrickscfg` de quem roda; o projeto não presume nenhum.

**Nunca escolher o perfil automaticamente.** Passe `--profile <nome>` em cada comando e deixe o usuário decidir.

### Workspace

Único SQL Warehouse: *Serverless Starter Warehouse* (2X-Small), que fica STOPPED fora de uso — sobe sozinho na primeira consulta, com alguns segundos de latência.

O projeto usa um **catálogo dedicado, `lakehouse_validator`**, com quatro schemas: `raw` (com o Volume onde o App deposita o arquivo aprovado), `bronze`, `silver` e `gold`. Não reaproveite os catálogos `lakehouse`, `lakehouse_tea` ou `lakehouse_rotaperfume` — pertencem a outros projetos. Ver [docs/decisoes.md](docs/decisoes.md) §6.7–6.8.

**O catálogo nasce pela UI, não pela CLI.** A conta usa *Default Storage*, e a plataforma só permite criar catálogos assim pelo Catalog Explorer: `databricks catalogs create` falha nas duas formas possíveis — sem storage root (*"Metastore storage root URL does not exist"*) e com o storage root da conta (*"Please use the UI to create a catalog with Default Storage"*). Schemas e Volume nascem pela CLI normalmente. Os passos estão em [docs/prompts/00-setup.md](docs/prompts/00-setup.md), armadilha 1 e item 3a.

Trabalho relacionado a Databricks passa pelas skills: carregue `databricks-core` (raiz) e depois a skill do produto — `databricks-apps-python` para o App Streamlit, `databricks-dbsql` para as camadas em SQL (elas rodam no warehouse pelo `scripts/executar_sql.py`, não por um pipeline), `databricks-synthetic-data-gen` para o dataset, `databricks-unity-catalog` para catálogos e tabelas.

As skills estão instaladas em três locais: `.agents/skills/` (completa, 40 skills, via `.ai-dev-kit`, profile `all`), `.github/skills/` (40) e `.claude/skills/` (8, só as usadas pelo projeto). `.agents/skills/` é a referência.

Nas skills, credenciais vêm do SDK (`Config()`) — nunca hardcoded. Em Databricks Apps, recursos são referenciados via `valueFrom` no `app.yaml`.

## Versionamento

O projeto é versionado em git. A versão construída na live e o histórico de ensaios ficam fora deste repositório.

O [.gitignore](.gitignore) reflete duas decisões já tomadas — não as reverta sem perguntar:

- **As skills Databricks são ignoradas** (`.agents/skills/`, `.github/skills/`, `.claude/skills/`). São ~6,8 MB em 636 arquivos instalados pela ferramenta, com as versões fixadas em `.ai-dev-kit/skills.lock`. Mantenha `.ai-dev-kit/` versionado e reinstale as skills pela ferramenta em vez de commitar as cópias.
- **Os 5 `.xlsx` de `data/output/` são versionados de propósito**, para a demo sair pronta sem gerar o dataset antes.

Segredos ficam em `.claude/settings.local.json` (ignorado). O `.claude/settings.json` é versionado e contém apenas `enabledPlugins` — tokens de API vão sempre para o arquivo local.
