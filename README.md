# AgnesFree

Análise profunda da **Agnes AI** (Sapiens AI): API multimodal compatível com OpenAI oferecendo texto (`agnes-2.0-flash`), imagem (`agnes-image-2.1-flash`) e vídeo (`agnes-video-v2.0`) a **US$ 0** — modelos, limites do plano gratuito, planos pagos, desempenho, privacidade e veredito prático.

**Guia completo:** https://inematds.github.io/agnesfree/guia/

**Testar / conhecer a Agnes AI (campanha oficial):** https://agnes-ai.com/campaign

## Notas de API — as dicas de quem usou

[`NOTAS-API.md`](NOTAS-API.md) é o documento vivo do que a doc **promete** vs. o que foi **observado** em teste real (~70 chamadas à API): parâmetros que funcionam, os que dão HTTP 400, retry/503, custo e cota, regras de prompt e o que **não** existe.

As 6 regras que valem dinheiro (detalhes no doc):

1. **Prompts em inglês** — português apanha do filtro de conteúdo e trava geração legítima com HTTP 400.
2. **Pedir explicitamente `ONE SINGLE bushy tail`** em animais com cauda — senão a pose frontal gera duas caudas.
3. **Retry com backoff é obrigatório** — ~34% das chamadas falham com 503, e o retry recupera ~100%.
4. **Descritor de estilo só com estética** — palavras como "fur" ou "children's book" injetam personagens em prompts de cenário.
5. **1K (32s) para volume** — 4K custa 153s e falha muito.
6. **Baixar o PNG imediatamente** — a URL é temporária.

Base da API: `https://apihub.agnes-ai.com/v1` (compatível com OpenAI), auth via `Authorization: Bearer $AGNES_API_KEY`.

Parte do ecossistema [INEMA.CLUB](https://inema.club).
