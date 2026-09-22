# Atualizador de slides: verifica legislação com IA e recria o deck na máscara nova

Workflow n8n de **21 nós** para um professor de cursos preparatórios que tem centenas de aulas antigas em PowerPoint. O professor solta o `.pptx` numa pasta do Google Drive; o workflow extrai o texto, **verifica cada lei, artigo, súmula e jurisprudência citada contra fontes oficiais** (Planalto, DOU, STF, STJ) usando Claude com busca na web, gera um relatório de divergências em Markdown, reescreve o roteiro completo de slides com o conteúdo atualizado e monta um novo `.pptx` no template atual da marca. O original vai para "Processados". Em produção desde agosto de 2026.

## Fluxo

```
Schedule ──► Listar pasta de entrada ──► só .pptx ──► Pegar o mais antigo
   ──► Drive API: copiar como Google Slides ──► export text/plain ──► apagar cópia temporária
   ──► Normalizar texto ──► Claude + web_search: verificação de legislação (JSON)
   ──► Validar ──► salvar Relatório_Atualizacao.md na entrada
   ──► Claude: roteiro completo de slides (JSON) ──► Validar
   ──► pptx-service (/gerar-pptx, máscara da marca) ──► upload do PPTX novo ──► mover original
```

## Detalhes que importam

- **Extração de texto sem biblioteca**: em vez de parsear o PPTX, o workflow copia o arquivo como Google Slides pela Drive API (`/copy` com `mimeType` de apresentação), exporta como `text/plain` e apaga a cópia. Três chamadas HTTP, zero dependência.
- **Verificação com regras rígidas no prompt**: usar `web_search` para cada dispositivo, priorizar fontes oficiais, **nunca corrigir de memória**; o que não for confirmado entra na lista como `nao_verificado` com gravidade baixa, em vez de ser "corrigido". Saída em JSON com `status` (`atualizado` / `com_divergencias`), lista de divergências (trecho do slide, situação, texto atual, fonte, gravidade) e `conteudo_atualizado`.
- **Relatório antes do deck**: o Markdown com as divergências é salvo antes de gerar os slides, e a nota do workflow deixa explícito que a **revisão humana do relatório é obrigatória** antes de usar o material.
- **Roteiro sem resumo**: o segundo prompt exige cobrir todo o conteúdo na mesma ordem e nível de detalhe ("a densidade vem da quantidade de slides, não do tamanho de cada slide"), com tipos de slide fechados (`agenda`, `secao`, `conteudo`, `definicao`, `comparativo`, `questao`, …) que o serviço de PPTX sabe renderizar.
- **Validação defensiva dos dois JSONs**: detecta truncamento por `max_tokens` com instrução de correção, extrai o JSON mesmo com texto em volta, valida campos obrigatórios e substitui travessões (regra editorial do cliente) em todas as strings recursivamente.
- **Processamento de um arquivo por execução**, sempre o mais antigo: fila natural sem estado extra; o que já foi movido não volta.

## Stack

n8n · Google Drive API v3 (copy/export) · Anthropic API (Claude com ferramenta `web_search`) · pptx-service próprio (Node.js, geração de PPTX a partir de JSON em template da marca)

## Como usar

1. Importe `workflow.json`.
2. Credenciais: Google Drive (OAuth2, usada nos nós de Drive e nas 3 chamadas HTTP) e Anthropic (header `x-api-key`).
3. Troque `ID_PASTA_ENTRADA`, `ID_PASTA_SAIDA` e `ID_PASTA_PROCESSADOS` pelos IDs reais.
4. Suba o `pptx-service` na mesma rede do n8n (`http://<projeto>_pptx-service:3000/gerar-pptx`).
5. Ative e coloque um `.pptx` na pasta de entrada.

## Placeholders

`ID_PASTA_ENTRADA`, `ID_PASTA_SAIDA`, `ID_PASTA_PROCESSADOS`, `CREDENTIAL_ID`.
