# Áudio de aula → material didático em PDF

Workflow n8n de **13 nós** que vigia uma pasta do Google Drive: o professor grava a aula no celular, solta o áudio na pasta, e em poucos minutos recebe na mesma pasta um **PDF diagramado** com o conteúdo organizado (seções, listas, destaques, questões) mais a transcrição em texto. Em produção desde setembro de 2026.

## Fluxo

```
Google Drive Trigger (pasta, a cada 2 min) ──► Config ──► Filtrar Áudio (tipo e limite de 25 MB)
   ──► Baixar ──► Preparar Áudio (.opus/.oga → .ogg) ──► Whisper (OpenAI, pt)
   ──► Montar Chamada IA ──► Claude (JSON do material) ──► Processar Material (JSON → HTML A4)
   ──► html-to-pdf (serviço próprio, Chromium) ──► Preparar PDF ──► Salvar PDF + transcrição no Drive
```

## O que cada etapa faz

- **Filtrar Áudio**: aceita só arquivos de áudio (por MIME ou extensão) e falha com mensagem clara quando o arquivo passa do limite do Whisper, dizendo ao professor como resolver ("salve em MP3 mono 48 kbps, ~70 min, ou divida"). O PDF vai para a mesma subpasta de onde o áudio veio.
- **Preparar Áudio**: renomeia `.opus`/`.oga` (formato padrão do WhatsApp) para `.ogg`, que o Whisper aceita, e garante extensão quando o arquivo vem sem.
- **Transcrever**: Whisper via HTTP multipart, idioma fixo `pt`, resposta em texto puro.
- **Montar Chamada IA**: rejeita transcrições com menos de 300 caracteres (áudio sem fala), decodifica o system prompt (guardado em base64 no `Config`) e monta a chamada com temperatura baixa. O prompt tem **regras de fidelidade**: o material vem do que o professor falou, sem inventar lei ou jurisprudência que ele não citou.
- **Processar Material**: parser tolerante do JSON (remove cercas, procura o primeiro `{...}`), detecta corte por `max_tokens` e avisa para dividir o áudio, valida título e seções, e renderiza o HTML A4 com um sistema de blocos (`paragrafo`, `lista`, `destaque`, `passos`, `questao`, `tabela`…) com o CSS da marca embutido. Marcações `**negrito**` e `*itálico*` são convertidas depois do escape de HTML, para nunca injetar tags vindas da IA.
- **Gerar PDF**: envia o HTML ao [ufem-screenshot](https://github.com/willianbrognoli/ufem-screenshot) (`/html-to-pdf`), que devolve base64; `Preparar PDF` transforma em binário e salva no Drive junto com um `.txt` da transcrição.

## Decisões de projeto

- **Drive como interface do usuário**: o professor não aprende ferramenta nova; pasta de entrada e saída são a mesma.
- **Erros que explicam o que fazer**: cada `throw` é escrito para quem vai ler o log sem saber n8n (tamanho do arquivo, áudio sem fala, resposta cortada).
- **HTML gerado por código, não pela IA**: a IA entrega estrutura em JSON; o layout é determinístico e consistente entre materiais.
- **Prompt em base64 no Config**: evita problemas de aspas e quebras de linha no JSON do nó Set e mantém o prompt versionado junto com o workflow.

## Stack

n8n · Google Drive API · OpenAI Whisper · Anthropic API (Claude) · serviço próprio HTML → PDF (Puppeteer/Chromium)

## Como usar

1. Importe `workflow.json`.
2. Credenciais: Google Drive (OAuth2), OpenAI (bearer), Anthropic (header).
3. No **Config** e no trigger: `FOLDER_ID` da pasta vigiada, `PDF_URL` do serviço de conversão, `LOGO_URL`, `SITE_URL`, `WHATSAPP_FMT`.
4. Ative e solte um áudio na pasta.

## Placeholders

`ID_DA_PASTA_DRIVE`, `SEU-SCREENSHOT-SERVICE`, `(DD) 9XXXX-XXXX`, `CREDENTIAL_ID`.
