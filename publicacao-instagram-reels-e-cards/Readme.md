# Publicação automática no Instagram: Reels (vídeo) e cards (imagem) a partir de uma fila em Data Table

Dois workflows n8n curtos que fecham o ciclo de pipelines de conteúdo maiores: tudo que já foi produzido e aprovado fica numa **Data Table do n8n** com status, e estes workflows publicam um item por vez, no horário certo, marcando o que saiu. Padrão de fila simples, sem banco externo.

| Arquivo | Nós | Gatilho | O que publica |
|---|---|---|---|
| `01-questao-do-dia-publicar-reels.json` | 11 | seg–sex, 9h–13h, a cada 30 min | 1 Reel (vídeo vertical gerado por outro pipeline) |
| `02-publicar-cards-instagram-facebook.json` | 8 | diário 09h | 1 card (imagem) no Instagram e na página do Facebook |

## 01. Questão do Dia: Reels

```
Cron ──► Config ──► Data Table: 1ª linha com status "pronto" (ordem por número)
   ──► Preparar Item (legenda com fallback) ──► Drive API: baixar MP4 ──► base64
   ──► Cloudinary: upload de vídeo (unsigned preset) ──► Graph API: container REELS
   ──► polling status_code até FINISHED (5 s × 60) ──► media_publish ──► Data Table: status "publicado" + ig_id
```

- **Fila em Data Table**: `status` (`pronto` → `publicado`), `numero` como ordem, `video_drive_id`, `legenda`, `data_pub`, `ig_id`. Quem produz o vídeo só precisa deixar a linha como `pronto`.
- **Legenda com fallback**: se a legenda gravada tiver menos de 50 caracteres, o código monta uma padrão com banca, órgão, chamada para comentar e hashtags. Nunca sai um Reel sem texto.
- **Cloudinary como ponte**: a Graph API exige `video_url` público; o Drive não serve para isso. O vídeo sobe por multipart em base64 com upload preset unsigned, sem chave secreta no workflow.
- **Polling explícito do processamento**: o Instagram processa o vídeo de forma assíncrona; o código consulta `status_code` e trata `ERROR`/`EXPIRED` como falha com o número da questão na mensagem.
- **Cron restrito à janela de engajamento** (dias úteis, manhã) e um item por execução: a fila esvazia no ritmo de 1 Reel a cada 30 min sem precisar de agendamento por item.

## 02. Cards: Instagram + Facebook

```
09h ──► Config (IG user id, page id) ──► Data Table: 1º card com publicado = false
   ──► IG: container (image_url + legenda) ──► espera 45 s ──► IG: media_publish
   ──► FB: /photos na página (mesma imagem e legenda) ──► Data Table: publicado = true + ids
```

- Usa o **nó nativo Facebook Graph API** do n8n com a credencial da conta, sem código: três chamadas (`/media`, `/media_publish`, `/photos`).
- Mesma imagem e legenda nas duas redes; os ids de mídia ficam gravados na linha para auditoria.
- Como o container de imagem processa em segundos, uma espera fixa de 45 s substitui o polling.

## Stack

n8n (Data Tables, Schedule, Wait) · Instagram Graph API (Reels e imagem) · Facebook Pages API · Google Drive API · Cloudinary

## Como usar

1. Importe os JSONs e crie as Data Tables (`questao_dia_controle`, `cards_educacao_vida_real`) com as colunas citadas acima.
2. Credenciais: Google Drive (OAuth2) no 01; Facebook Graph (token de página com `instagram_content_publish` e `pages_manage_posts`) no 02.
3. Preencha o nó **Config** de cada um (`igUserId`, `pageToken`/`pageId`, `cloudName`, `uploadPreset`).
4. Ative.

## Placeholders

Os dois workflows já vieram com placeholders nos campos de conta (`PREENCHER_IG_USER_ID`, `PREENCHER_PAGE_ID`, `<__PLACEHOLDER_VALUE__…>`); no repo entram também `CLOUDINARY_CLOUD`, `ID_DATA_TABLE` e `CREDENTIAL_ID`.
