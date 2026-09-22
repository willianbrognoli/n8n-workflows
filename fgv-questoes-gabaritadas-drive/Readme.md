# Provas da FGV → banco de questões gabaritadas (JSON + imagens no Drive)

Workflow n8n de **26 nós** que, a partir do slug de um concurso no site público da banca FGV, baixa as provas e o gabarito oficial em PDF, **usa o Claude para ler os PDFs diretamente** (sem OCR próprio), cruza cada questão com a resposta certa do cargo/tipo correspondente e entrega no Google Drive um JSON estruturado por prova, mais o PNG das páginas que têm figuras. É a etapa de alimentação de um banco de questões usado na produção de apostilas.

## Fluxo

```
Manual ──► Config (slug, cargo, máx. provas, pasta, modelo)
   ──► GET página do concurso ──► Extrair Links (HTML → provas por cargo/tipo + gabarito)
   ──► Baixar gabarito PDF ──► base64 ──► Claude: gabarito em JSON {cargo-tipo: {nº: letra}}
   ──► Loop por prova
        ├──► Baixar prova PDF ──► upload do PDF fonte no Drive
        ├──► Claude: extrai TODAS as questões em JSON (disciplina, assunto, texto de apoio, enunciado, alternativas, tem_imagem, página)
        ├──► Montar JSON Final: casa com o gabarito, gera ids, propaga disciplina ──► upload JSON
        └──► Tem imagens? ──► Stirling: PDF → PNG (zip) ──► descompacta ──► só páginas com figura ──► upload PNGs
```

## Detalhes que importam

- **Scraping do HTML com um único regex de tokens**: percorre títulos (`<strong>`, `<h3>`) e links `<a>` em ordem, guarda o último título como "cargo corrente", e classifica cada PDF pelo texto do link e pelo nome do arquivo: `gabarito` (preferindo o definitivo), `Tipo N` (prova), e uma lista de exclusão para editais, recursos, resultados, redação etc. Suporta filtro por cargo e limite de provas no Config.
- **PDF direto na API do Claude** (`type: document`, base64, `max_tokens: 50000`): a prova em duas colunas e a tabela do gabarito são interpretadas pelo modelo. O prompt do gabarito descreve o formato da tabela da FGV (linha `Cargo - TIPO N`, números, letras, `*` para anulada) e exige JSON puro.
- **Casamento cargo × tipo tolerante**: normaliza acentos e pontuação dos dois lados e aceita match exato ou por inclusão (`nk.includes(cargo)`), porque o nome do cargo no gabarito e no link da prova raramente batem letra por letra.
- **Ids estáveis**: `fgv-<slug>-<cargo>-t<tipo>-q<n>` para cada questão, permitindo deduplicar entre execuções e referenciar imagens (`<provaId>-p<página>.png`).
- **Imagens só quando precisa**: o modelo marca `tem_imagem` e a página; o Stirling converte o PDF inteiro em PNG (zip), e o código descarta tudo que não está na lista de páginas. Sem figura, o ramo nem roda.
- **Falhas explícitas**: `stop_reason === 'max_tokens'` derruba a execução com a instrução de dividir o PDF, em vez de gravar JSON incompleto.

## Stack

n8n · Anthropic API (Claude com entrada de PDF) · Google Drive · Stirling PDF (`/convert/pdf/img`) · HTTP scraping

## Como usar

1. Importe `workflow.json`.
2. Credenciais: Anthropic (Header Auth `x-api-key`) e Google Drive (OAuth2).
3. No **Config**: `slug` do concurso (o final da URL em `conhecimento.fgv.br/concursos/…`), `filtroCargo` (vazio = todos), `maxProvas` (0 = todas), `driveFolderId`, `stirlingUrl`, `anthropicModel`.
4. Execute manualmente. Saída no Drive: `<slug>-<prova>.pdf`, `<provaId>.json` e `<provaId>-p<N>.png`.

## Placeholders

`ID_DA_PASTA_DRIVE`, `CREDENTIAL_ID`. O host do Stirling é interno do Docker e ficou como está.
