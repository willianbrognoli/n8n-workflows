# Página de curso no blog + carrossel e story no Instagram com imagens geradas por IA

Workflow n8n de **44 nós** para uma instituição de ensino a distância. A cada 6 horas pega o próximo curso pendente de uma planilha, pesquisa a profissão na web, escreve uma página de blog completa sobre mercado de trabalho e publica no WordPress com capa gerada por IA; depois monta um carrossel institucional de 7 slides (1080×1350) e um story (1080×1920) com **fotos de ambiente geradas por IA por slide**, e publica os dois no Instagram. É a versão mais elaborada da família de workflows de conteúdo deste repositório (compare com o [01](../01-blog-academia-de-policia)).

## Fluxo

```
Schedule 6h ──► Config ──► Planilha (próximo curso sem "ok")
   ──► Pesquisa profunda (dossiê de mercado: YouTube, Reddit, fóruns, vagas) ──► Claude: página HTML
   ──► WordPress post ──► prompt de capa ──► imagem IA ──► Cloudinary ──► TinyPNG ──► WebP ──► featured image
   ──► marcar "ok"
   ──► Claude: roteiro do carrossel (JSON) ──► 1 prompt de imagem por slide ──► imagens IA ──► Cloudinary
   ──► Gerar HTMLs Slides v3 (template por tipo de curso) ──► screenshot service ──► Instagram carrossel
   ──► container do story ──► polling de status ──► publicar story
```

## Detalhes que importam

- **Dois templates de carrossel por tipo de curso**, detectado pela URL: tecnólogo (hero / números / o que aprende / atalho 1 / atalho 2 / combinação / CTA) e pós-graduação (hero / números / o que aprende / concurso / progressão / validade / CTA). O HTML dos slides é gerado em código (28 KB de template) com o sistema de design da marca; a IA só fornece os textos.
- **Imagens de ambiente, sem pessoas**: o prompt base de geração proíbe rostos, mãos, texto e logos ("photorealistic editorial photograph, no people…") para evitar a aparência artificial típica de IA em fotos de gente. Cada slide recebe um prompt temático da profissão; a capa do blog outro. O gerador de imagem é configurável (o código tem suporte a Pollinations com dimensões por proporção, e o nó HTTP atual chama a API de imagens da OpenAI).
- **Pesquisa antes da escrita**: o dossiê de mercado pede fontes reais e verificáveis, com instruções explícitas para não inventar dados; o artigo e o carrossel só usam o que a pesquisa trouxe.
- **Parser de JSON balanceado**: `Processar Roteiro` extrai o primeiro objeto `{…}` respeitando strings e escapes, resolvendo o caso do modelo escrever texto depois do JSON.
- **Story via Graph API com polling**: cria o container `media_type: STORIES`, consulta `status_code` a cada 3 s (até 20 tentativas) e só então publica; erros trazem o payload completo para diagnóstico.
- **Cloudinary como buffer** entre a geração de imagem e o consumo pelos templates e pelo Instagram (que exige URL pública).

## Stack

n8n · Anthropic API (Claude) · pesquisa via API de respostas com busca web · geração de imagens por IA · Cloudinary · WordPress REST + Rank Math · TinyPNG · serviço próprio de screenshot · Instagram Graph API

## Como usar

1. Importe `workflow.json`.
2. **Mova as chaves para credenciais do n8n** antes de ativar (ver nota abaixo): Anthropic, provedor de pesquisa e imagem, TinyPNG, Google Sheets, Instagram.
3. No **Config**: `WORDPRESS_URL`, `GOOGLE_SHEETS_URL` (colunas `Nome do Curso`, `link do curso no site`, `status`), prompt do sistema.
4. Em `Criar Container Story`: `IG_USER_ID` e token.
5. Ative.

## Placeholders

`ANTHROPIC_API_KEY_AQUI`, `META_PAGE_TOKEN_AQUI`, `IG_USER_ID`, `ID_DA_PLANILHA`, `CLOUDINARY_CLOUD`, `CLOUDINARY_PRESET`, `SEU-SCREENSHOT-SERVICE`, `55DDNUMERO`.

> Nota de segurança: esta é uma versão mais antiga do padrão, em que a chave da Anthropic ficava no nó `Config` e o token da página da Meta dentro de um nó de código. Os workflows posteriores deste repositório (01, 02) já usam credenciais do n8n para tudo; ao reaproveitar este, faça o mesmo.
