# Blog + Instagram automáticos com IA (WordPress)

Workflow n8n de **68 nós** que mantém sozinho o blog e o Instagram de uma escola de cursos preparatórios: pesquisa, escreve, publica no WordPress com SEO, gera a capa, transforma o artigo em carrossel e publica no Instagram, e ainda renova a própria pauta toda semana. Em produção desde abril de 2026, publicando 2 artigos por dia (1 de tema, 1 de notícia) e 1 carrossel a cada 2 horas de conteúdo pendente.

## Os quatro fluxos

```
08h  Tema      Planilha "Temas" ──► Perplexity (pesquisa) ──► Claude (artigo HTML)
                                                                     │
15h  Notícias  4 feeds RSS ──► filtro regex 48h ──► Claude escolhe ──┤
                                                                     ▼
                       WordPress (post + Rank Math SEO + schema JSON-LD)
                                    │
                       Capa: HTML ──► screenshot ──► TinyPNG ──► WebP ──► featured image
                                    │
8h-22h/2h  Instagram   Post sem tag "ig-ok" ──► Claude (roteiro) ──► slides HTML ──► screenshot
                       ──► upload WP ──► Graph API (containers + carrossel) ──► tag "ig-ok"

Dom 19h  Pauta         Temas atuais ──► Perplexity (tendências) ──► Claude (8 temas novos) ──► planilha
```

### 1. Artigo de tema (diário, 08h)
Lê a planilha Google Sheets, pega a primeira linha sem status `ok`, monta uma pesquisa profunda no **Perplexity** com limites explícitos (sem inventar lei, número de portaria, julgado ou salário; jurisprudência só STF/STJ com número e ano), envia o dossiê ao **Claude** com um system prompt editorial (guardado em base64 no nó Config) e recebe o artigo em HTML.

### 2. Artigo de notícia (diário, 15h)
Puxa 4 feeds RSS (Google News com duas buscas, Gran Cursos, Estratégia), parseia o XML em código, filtra por regex o que é do nicho e tem sinal de edital/andamento nas últimas 48 h, deduplica por título, e pede ao Claude que **escolha a notícia mais relevante** devolvendo JSON (índice, UF, cidade, fase, título e palavra-chave). Daí segue o mesmo caminho do artigo de tema.

### 3. Publicação no WordPress
O código `Extrair HTML` limpa o retorno da IA (remove `<style>` e `<script>` que não sejam JSON-LD), separa os blocos de schema, envolve tudo num bloco `wp:html` e monta os campos do **Rank Math** (focus keyword, title, description). Categoria e tag de UF são resolvidas por mapa de IDs no Config. A capa é um HTML de card renderizado pelo [ufem-screenshot](https://github.com/willianbrognoli/ufem-screenshot), comprimida no TinyPNG, convertida para WebP, enviada à biblioteca de mídia com alt text e definida como imagem destacada. No fim, marca `ok` na planilha e dispara um webhook para o fluxo do Instagram.

### 4. Carrossel no Instagram (a cada 2 h, 8h-22h)
Busca os 10 posts mais recentes na REST API do WordPress, escolhe o primeiro das categorias-alvo publicado há até 5 dias e **sem a tag `ig-ok`** (a tag é a trava contra republicação; o nó falha de propósito se o ID dela não estiver configurado). O Claude gera o roteiro em JSON com regras diferentes para notícia (4 slides, manchete com cidade e dado forte) e tema (7 slides). Os slides são HTML com o sistema de design da marca, renderizados em 1080×1080 e um story em 1080×1920, cortados, enviados ao WordPress (que serve como CDN) e publicados via **Instagram Graph API** em três passos: containers das imagens, container do carrossel, publish. Ao final aplica a tag `ig-ok` no post.

### 5. Pauta semanal (domingo, 19h)
Lê os temas já existentes, pede ao Perplexity as tendências do nicho, pede ao Claude 8 temas novos em JSON com categoria fixa (Legislação, Disciplinas, Preparação, Carreira) e palavra-chave, deduplica contra o que já existe (normalizando acentos) e adiciona as linhas na planilha. O fluxo 1 nunca fica sem pauta.

## Decisões de projeto

- **Tudo configurado em nós `Config` (Set em modo raw JSON)**: URL do site, IDs de categoria/tag, planilha, serviço de screenshot. Trocar de cliente é editar 4 nós.
- **Chamadas de IA por HTTP Request, não pelos nós nativos**: controle total de `system`, `max_tokens`, parse do retorno e retry.
- **Parse defensivo do JSON da IA**: remove cercas de código e, se falhar, procura o primeiro `{...}` ou `[...]` do texto antes de lançar erro com trecho da resposta.
- **Idempotência por marcadores** (status `ok` na planilha, tag `ig-ok` no post): reexecutar o workflow nunca duplica publicação.
- **Encerramento silencioso**: quando não há pauta ou post pendente, o nó de escolha devolve `[]` e a execução termina sem erro.
- **WordPress como CDN das imagens do Instagram**: a Graph API exige URLs públicas; subir na biblioteca de mídia evita mais um serviço.

## Stack

n8n · Anthropic API (Claude) · Perplexity API · WordPress REST API + Rank Math · Google Sheets · Instagram Graph API · TinyPNG API · serviço próprio de screenshot (Puppeteer) · Evolution API (WhatsApp, opcional)

## Como usar

1. Importe `workflow.json` no n8n (Workflow → Import from File).
2. Crie as credenciais: Anthropic (header `x-api-key`), Perplexity (bearer), WordPress (basic auth com application password), Google Sheets (OAuth2), TinyPNG (basic auth), Instagram (header `Authorization: Bearer`).
3. Edite os nós **Config**, **Config Notícias**, **Config IG** e **Config Pauta**: `WORDPRESS_URL`, `GOOGLE_SHEETS_URL`, IDs de categorias/tags, `SCREENSHOT_URL`, `IG_USER_ID`.
4. Planilha com a aba `Temas` e colunas `tema`, `categoria`, `foco`, `palavra_chave`, `status`.
5. Ative. Para testar sem publicar, mude `WP_STATUS` para `draft`.

## Placeholders no JSON

`ID_DA_PLANILHA`, `IG_USER_ID`, `SEU-SCREENSHOT-SERVICE`, `SEU-N8N`, `55DDNUMERO`, `INSTANCIA_EVOLUTION`, `CREDENTIAL_ID`. Nenhuma chave de API está no arquivo; todas ficam em credenciais do n8n.
