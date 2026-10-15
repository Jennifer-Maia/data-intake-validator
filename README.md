# Data Intake Validator

Solução demonstrativa de Engenharia de Dados. Planilhas Excel de áreas diferentes da empresa são
validadas contra um contrato de dados antes de a ingestão permitir a entrada. O que passa percorre o
lakehouse — Bronze, Silver e Gold — até virar consumo.

```
Usuário → Validação → Ingestão → Bronze → Silver → Gold → Consumo
```

## O princípio

**A IA reduz o trabalho de implementação, mas não substitui a definição das regras — e é por isso que
este projeto começa por um contrato, não por código.**

- **A validação não depende do App.** É um módulo Python puro; o App só renderiza o resultado. A mesma
  validação serve a qualquer mecanismo de ingestão.
- **O sistema nunca corrige dado em silêncio.** O arquivo é aprovado inteiro ou rejeitado inteiro. As
  sugestões são explícitas, e a decisão fica com uma pessoa.
- **Nenhuma regra vive fora do contrato.** As 22 decisões de projeto estão em `docs/decisoes.md`, cada
  uma com sua justificativa.

## Estrutura

| Caminho | O que é |
| --- | --- |
| `docs/projeto.md` | a especificação: objetivo, schema, escopo e critérios de conclusão |
| `docs/decisoes.md` | as 22 decisões sobre os pontos em aberto, com justificativa |
| `docs/prompts/` | os prompts que constroem o projeto, na ordem em que são usados |
| `scripts/generate_sales.py` | gera o dataset e os cinco arquivos de teste |
| `scripts/executar_sql.py` | executa um statement SQL no warehouse da workspace |
| `scripts/verificar_estado_live.py` | checklist do ponto de partida, antes de cada execução |
| `data/output/` | os cinco `.xlsx` de teste: dois aprovados e três rejeitados |
| `validator/`, `apps/`, `ingestion/`, `notebooks/` | onde a construção acontece |
| `.ai-dev-kit/` | versões fixadas das skills de Databricks usadas no desenvolvimento |

## Estado deste repositório

Este é o **ponto de partida**: especificação, decisões, gerador, arquivos de teste e a sequência de
prompts. O contrato, o módulo de validação, o App e as camadas são construídos a partir de
`docs/prompts/`. A versão construída não faz parte deste repositório.

## Como rodar

```bash
python -m pip install -r requirements.txt
python scripts/generate_sales.py
python scripts/verificar_estado_live.py --profile <perfil>
```

O perfil do Databricks CLI é sempre escolhido por quem roda. Nenhum script o presume.
