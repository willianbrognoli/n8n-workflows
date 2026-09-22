# Captação de leads e vendas: webhooks, Elementor e plataforma de cursos → Sheets + PostgreSQL

Três workflows n8n que formam a camada de **ingestão** do dashboard de marketing de uma professora de cursos jurídicos (o mesmo banco que o [dashboard-fauth](https://github.com/willianbrognoli/dashboard-fauth) lê e que o [workflow 06](../06-meta-ads-coleta-diaria-metricas) alimenta com mídia paga). Juntos, eles fazem todo lead e toda venda cair com UTMs normalizadas em duas camadas: Google Sheets para a equipe e PostgreSQL para o BI. Em produção desde julho de 2026.

| Arquivo | Nós | Gatilho | Função |
|---|---|---|---|
| `01-leads-site-5-webhooks.json` | 8 | 5 webhooks | Recebe formulários de 5 origens do site, normaliza e grava |
| `02-tutory-captura-de-vendas.json` | 5 | webhook | Recebe eventos de pedido da plataforma de cursos e faz upsert de compras |
| `03-elementor-para-planilha-diario.json` | 11 | diário 00:05 | Coleta os envios do Elementor Forms do dia anterior via endpoint próprio no WordPress |

## 01. Leads do site (5 webhooks, 1 código)

Cinco caminhos de webhook (`compra`, `blog`, `whatsapp`, `amostra`, `materiais-youtube`) entram no **mesmo nó de código**, que descobre a origem pela URL chamada e escolhe a planilha de destino num mapa. O ponto central é o **parser agnóstico de formulário**: ele achata qualquer payload (`fields`, `values`, `form_fields`, arrays `{key, value}`, query string) num dicionário chave→valor e depois procura e-mail, telefone, nome e UTMs **por regex nas chaves e nos valores**, sem depender de nome de campo fixo. Um mesmo workflow atende formulários de plugins diferentes sem ajuste por formulário.

Cada lead vira uma linha na planilha da origem e um `INSERT` no Postgres (`leads`) com data/hora em `America/Sao_Paulo`, origem, formulário, UF, UTMs, `gclid` e `fbclid`.

## 02. Vendas (webhook da plataforma de cursos)

1. **Grava o evento bruto primeiro** (`webhook_eventos`, com `payload` e `headers` em JSONB; a tabela é criada com `CREATE TABLE IF NOT EXISTS` na própria execução). Se o parser mudar, dá para reprocessar tudo depois.
2. `Prepara compra` filtra só eventos de pedido (`pix_gerado`, `boleto_gerado`, `pagamento_aprovado`, `recusado`, `cancelado`), mapeia status, separa nome/sobrenome, normaliza telefone para E.164 brasileiro, converte estado por extenso em UF (sem acento) e resolve o valor entre três campos possíveis do payload.
3. **Upsert em `compras` por `transacao_id`**: o mesmo pedido chega várias vezes (PIX gerado → pago); a linha é atualizada, nunca duplicada. Status `paid` é o que o dashboard usa como receita.

## 03. Elementor → planilha (diário)

O Elementor Pro guarda os envios de formulário no banco do WordPress, mas não expõe REST nativo. Um mini-plugin no site (`/wp-json/n8n-fix/v1/submissions`, protegido por token no header) expõe os envios por período. O workflow:

- Define o período em `America/Sao_Paulo` (ontem em produção; hoje quando disparado à mão) e converte para GMT porque o WordPress grava em UTC.
- Normaliza cada envio com o mesmo parser agnóstico do workflow 01 (mais `ttclid` e `msclkid`), resolvendo a data a partir de `created_at_gmt` ou `created_at`.
- **Deduplica contra a coluna `submission_id`** já gravada na planilha (lê só a coluna `P:P` para ser leve) antes de inserir.
- Grava na aba `Elementor` da planilha do dia (criada se não existir), na base geral de `Leads`, ordena a aba por formulário via Sheets API `batchUpdate`, e insere no Postgres.

## Decisões de projeto

- **Duas camadas de escrita, uma fonte de verdade**: planilha para a equipe olhar e editar; Postgres para o dashboard. O Postgres é o que vale para métricas.
- **Evento bruto antes do parse**: a tabela `webhook_eventos` é o seguro contra mudanças de payload da plataforma.
- **Idempotência em cada caminho**: `ON CONFLICT` por transação nas vendas, `submission_id` nos formulários, e o modo de execução (manual vs produção) controlando o período para reprocessamento seguro.
- **Parser por regex nas chaves**: um único nó atende formulários com nomes de campo diferentes; o custo é um pouco de código a mais, o ganho é zero manutenção quando o cliente cria um formulário novo.

## Stack

n8n · Webhooks · Google Sheets API (append + batchUpdate) · PostgreSQL (JSONB, upsert) · WordPress REST (endpoint customizado) · Luxon

## Como usar

1. Importe os três JSONs.
2. Credenciais: Google Sheets (OAuth2), Postgres, e no 03 um Header Auth com o token do endpoint do WordPress.
3. Troque os IDs de planilha (`ID_PLANILHA_*`) e, no 03, a URL do site.
4. Crie as tabelas `leads` e `compras` (colunas visíveis nas queries dos nós PG) com índice único em `compras.transacao_id`.
5. Aponte os formulários do site para `https://SEU-N8N/webhook/<path>` e o webhook da plataforma de cursos para `/webhook/tutory-fauth`.

## Placeholders

`ID_PLANILHA_COMPRA`, `ID_PLANILHA_BLOG`, `ID_PLANILHA_WHATSAPP`, `ID_PLANILHA_AMOSTRA`, `ID_PLANILHA_YOUTUBE`, `ID_PLANILHA_ELEMENTOR`, `ID_PLANILHA_LEADS`, `CREDENTIAL_ID`. A URL pública do site do cliente foi mantida no 03 (é o endpoint chamado).
