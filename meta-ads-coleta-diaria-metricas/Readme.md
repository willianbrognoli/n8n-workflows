# Meta Ads → PostgreSQL + Google Sheets: coleta diária de métricas

Workflow n8n de **14 nós** que todo dia, às 04:30, busca na Marketing API da Meta as métricas do dia anterior em três recortes (por campanha, por idade/gênero e por região/UF), normaliza conversões, e grava em duas camadas: **Google Sheets** (para o cliente abrir e ver) e **PostgreSQL com upsert idempotente** (para alimentar o [dashboard-fauth](https://github.com/willianbrognoli/dashboard-fauth)). Em produção desde julho de 2026.

## Fluxo

```
04:30 ──► Define período (ontem em produção; 30 dias em teste manual)
   ├──► Meta insights level=campaign ──► Trata ──► Sheets "Campanhas" ──► PG upsert metricas_campanhas
   ├──► Meta insights breakdowns=age,gender ──► Trata ──► Sheets "Demografia" ──► PG upsert metricas_demografia
   └──► Meta insights breakdowns=region ──► Trata (região → UF) ──► Sheets "Regiao" ──► PG upsert metricas_regiao
```

## Decisões que valem registrar

- **Período depende do modo de execução**: `$execution.mode !== 'production'` carrega os últimos 30 dias; em produção pega só ontem. Testar na interface faz backfill automático sem precisar mudar nada.
- **Conversões por lista de prioridade, não por soma**: a Meta devolve o mesmo evento em vários `action_type` (`lead`, `onsite_conversion.lead_grouped`, `offsite_conversion.fb_pixel_lead`…). Somar tudo infla o número; o código percorre a lista em ordem e usa o primeiro que existir. Mesma lógica para compras (`omni_purchase`, `purchase`, pixel) e para o valor em `action_values`.
- **Região → UF por dicionário normalizado**: a API devolve nomes de estado em português ou inglês ("Federal District", "Parana"); o código remove acentos e parênteses e mapeia para a sigla, com `coalesce` para não perder linhas que não casam.
- **Upsert com `ON CONFLICT`** nas chaves naturais (`data + plataforma + campanha_id`, `data + plataforma + idade + genero`, `data + plataforma + regiao`): rodar duas vezes o mesmo dia atualiza em vez de duplicar; a planilha, em `append`, é só leitura para o cliente.
- **Números formatados em duas versões**: com vírgula e duas casas para a planilha (`Gasto`), e numéricos crus (`_gasto`, `_imp`) para o banco, no mesmo item.
- **`time_increment=1` e `limit=500`**: uma linha por dia por dimensão, sem paginação para uma conta de porte médio; a coluna `plataforma` já prevê Google Ads na mesma tabela.

## Stack

n8n · Meta Marketing API v23 (`/insights`) · Google Sheets · PostgreSQL (upsert) · Luxon para fuso `America/Sao_Paulo`

## Como usar

1. Importe `workflow.json`.
2. Credenciais: Facebook Graph API (token com `ads_read`), Google Sheets (OAuth2), Postgres.
3. Troque `act_ID_DA_CONTA_DE_ANUNCIOS` nas 3 chamadas e `ID_DA_PLANILHA` nos 3 nós de Sheets (abas `Campanhas`, `Demografia`, `Regiao`).
4. Crie as tabelas `metricas_campanhas`, `metricas_demografia` e `metricas_regiao` com as constraints únicas usadas no `ON CONFLICT` (o DDL está nas queries dos nós PG).
5. Execute manualmente uma vez para o backfill de 30 dias; depois ative.

## Placeholders

`act_ID_DA_CONTA_DE_ANUNCIOS`, `ID_DA_PLANILHA`, `CREDENTIAL_ID`.
