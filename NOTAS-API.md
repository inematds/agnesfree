# Agnes AI — notas de API (documento vivo)

Registro do que a documentação **promete** vs. o que foi **observado** em teste real.

- Sessão 1: 2026-07-17 · ~70 chamadas reais à API
- Fontes: [guia INEMA](https://inematds.github.io/agnesfree/guia/) · **doc oficial** https://wiki.agnes-ai.com/en/docs/agnes-image-21-flash
- Chave: `AGNES_API_KEY` em `.env` (gitignored). Base: `https://apihub.agnes-ai.com/v1`

**Convenção:** 📘 documentado, não verificado · ✅ observado e confirmado · ⚠️ observado e **contradiz** o documentado · ❌ **hipótese minha que caiu** · ❓ não testado

---

## 0. Leia isto primeiro — as 6 regras que valem dinheiro

1. **Prompts em INGLÊS.** Não por qualidade (empatada) — porque o **português apanha do filtro de conteúdo** e trava geração legítima com HTTP 400.
2. **Sempre pedir `ONE SINGLE bushy tail`** em qualquer animal com cauda. Sem isso, pose frontal gera **duas caudas**.
3. **Retry com backoff é obrigatório.** ~34% das chamadas falham com 503; o retry recupera ~100%.
4. **Descritor de estilo só com estética.** Palavras como "fur", "expressive eyes", "children's book" **injetam personagens** em prompts de cenário.
5. **1K (32s) para volume.** 4K custa 153s e falha muito.
6. **Baixar o PNG imediatamente** — a URL é temporária.

---

## 1. Parâmetros — a forma correta (doc oficial + confirmado)

```bash
curl https://apihub.agnes-ai.com/v1/images/generations \
  -H "Authorization: Bearer $AGNES_API_KEY" -H "Content-Type: application/json" \
  -d '{"model":"agnes-image-2.1-flash","prompt":"...","size":"1K","ratio":"16:9",
       "extra_body":{"response_format":"url"}}'
```

| Parâmetro | Tipo | Notas |
|---|---|---|
| `model` | string | `agnes-image-2.1-flash` |
| `prompt` | string | **em inglês** (ver §4) |
| `size` | string | `1K`..`4K`. Aceita `1024x768` legado, mas normaliza |
| `ratio` | string | `1:1`,`3:4`,`4:3`,`16:9`,`9:16`,`2:3`,`3:2`,`21:9`. Default `1:1` |
| `image` | string[] | **só dentro de `extra_body`** — img2img |
| `return_base64` | bool | t2i em base64 |
| `extra_body.response_format` | string | `url` \| `b64_json` ✅ funciona |

⚠️ **`response_format` no nível raiz → HTTP 400.** A doc avisa; confirmado.
✅ **Params desconhecidos são descartados em silêncio** (litellm `drop_params`) — **HTTP 200 não prova que o parâmetro foi usado**.

### ❌ NÃO existe
- **`seed`** — não está na doc. (Meu teste foi inconclusivo: 503 + 422, nenhuma imagem. Mas a doc não lista.)
- **LoRA / fine-tune** — não é plataforma de treino.

---

## 2. Custo, cota e limites

**Não há créditos. Resolução não custa nada a mais.** US$ 0 em qualquer tamanho. ✅ `usage` volta sempre zerado.

### ⚠️ Não existe comando de cota real
```bash
curl -s https://apihub.agnes-ai.com/v1/dashboard/billing/subscription -H "Authorization: Bearer $AGNES_API_KEY"
# {"soft_limit_usd":100000000,"hard_limit_usd":100000000,...}  <- valores de preenchimento (default one-api)
curl -s https://apihub.agnes-ai.com/v1/dashboard/billing/usage -H "Authorization: Bearer $AGNES_API_KEY"
# {"total_usage":0}  <- sempre 0, porque tudo é grátis. Não mede nada.
```
Servem só como **teste de saúde da chave**. Rotas admin do one-api (`/api/user/self`, `/key/info`) → 403 (WAF).
**O único medidor real de saturação é a taxa de 503.**

### 📘 Limites do plano gratuito (⚠️ NÃO confirmados — ver §3.1)
Texto 20 req/min · Imagens 1K 20/min · 2K 10/min · 3K-4K 1/min · Vídeo 1 req/min. Sem SLA.

### 📘 Token Plans (preços de fonte da comunidade — confirmar no checkout)
Starter US$4 (1.5k texto/5h) · Plus US$10 (7.5k) · Pro US$50 (30k). Todos: 4.000 imagens/dia, 100 RPM 1K, 80 RPM 2K.
O pago compra **velocidade**, não créditos de imagem. 📘 free e pago usam pools separados ❓.

---

## 3. ⚠️ Contradições e ❌ hipóteses derrubadas

### 3.1 ⚠️ O limite de "1 img/min" em 3K/4K não se manifestou
Duas chamadas 4K consecutivas: ambas aceitas. **Nenhum 429, nenhum header de quota.**
Amostra: 2. Não conclui "não há limite" — só que não observei enforcement.

### 3.2 ⚠️ O custo do 4K é tempo e falha, não cota
| Chamada | Resultado | Tempo |
|---|---|---|
| 4K (1ª) | **HTTP 503** | 50s |
| 4K (2ª) | 200 OK | **153s** |
| 1K | 200 OK | 32s |

4K real: 5248×2944, **23 MB** ✅ (bate com a tabela). Mas 23 MB > 10 MB ⇒ **4K não serve como referência** (§5).

### 3.3 ❌ "O modelo não sabe fazer texto" — ERRADO, eu generalizei de 1 amostra ruim
- ✅ Título do conto: **"O ESQUILO QUE ENCONTROU SUU AMIGO VERDADEIRO"** — 1 erro só (`SUU`/`SEU`), letra limpa.
- ✅ Placa: **"FLORESTA"** perfeito.
- ✅ Em EN, **"WELCOME TO THE FOREST"** apareceu **espontâneo** e legível.
- ❌ O caso que me enganou era um dashboard com dezenas de rótulos minúsculos.

**Regra real: densidade, não idioma.** Texto curto em letra grande ✅ · texto pequeno e denso ❌.
**Revisão humana obrigatória** — erros tipo `SUU` passam batido.

### 3.4 ❌ "castanha→muffin é falha de tradução do PT" — HIPÓTESE MINHA, NÃO CONFIRMADA
Na matriz, "castanha" virou **cupcake em 7/10 estilos**. Apostei em erro de tradução.
**O teste PT/EN não sustentou:** o PT entregou castanha correta.
**Erro de método meu:** escolhi o estilo Pixar pro teste — o único que **já acertava em PT**. Testei no caso que não podia falhar.
**Correlação real: ESTILO, não idioma.** Acertam: Pixar, aquarela. Erram: realista, futurista, anime, desenho-2d, flat, óleo, isométrico. ❓ Causa aberta.

### 3.5 ❌ Meus scripts imprimiram 5 vereditos falsos — leia o erro cru, não a conclusão
| Veredito falso | Realidade |
|---|---|
| "SEED FUNCIONA? NAO" | as 2 chamadas falharam (503/422). Nada foi provado |
| campos `image_url`/`init_image` "aceitos" | litellm **descartou** os params. HTTP 200 ≠ usado |
| "quebrou em n=2" (array) | era o teto de **10 MB/imagem** — a ref era a 4K de 23 MB |
| "quebrou em n=6" | foi **timeout meu de 300s**, não limite da API |
| "MONTAGENS_OK" | ImageMagick não existe na máquina; `echo` rodou assim mesmo |

**Padrão: a mensagem de erro crua da API sempre estava certa; a conclusão automática, errada.**

---

## 4. ⚠️ Idioma do prompt: use INGLÊS

| Idioma | Chamadas | Falhas/retries |
|---|---|---|
| Português | 5 | **9** (1 abandonado) |
| Inglês | 5 | **0** |

**O motivo decisivo — o filtro de conteúdo:**
- PT "um esquilo usando um galho comprido como alavanca para levantar uma pedra pesada" → **HTTP 400, 4× seguidas**.
- EN "a squirrel using a long stick as a lever to lift a heavy rock" → **OK de primeira**, execução perfeita.
- PT "letras formando a palavra FIM sobre um pôr do sol" → **`content_policy_violation`**.

400 é determinístico — não é fila. ⚠️ **Viés admitido:** no meu teste o PT sempre rodou **antes** do EN, então a diferença de *retry* pode estar inflada por ordem. O 400 repetido não.

**O EN não ganha em tudo:** na contagem, PT deu 1 esquilo + 1 placa (certo); **EN duplicou a placa**.

---

## 5. img2img (`extra_body.image`)

| Fato | Status |
|---|---|
| Campo correto | `extra_body.image` (array) ✅ — os outros nomes são descartados |
| Quantidade aceita | **1–5** ✅ · 6+ ❓ (meu timeout, não limite) |
| Quantidade **ÚTIL** | ⚠️ **NO MÁXIMO 2** — ver §5.1 |
| Tamanho | **máx. 10 MB por imagem** ✅ (msg de erro) ⇒ 4K rejeitada |
| Custo | ~7s por imagem extra no array |
| Velocidade | **2× mais rápido** que t2i (23–27s vs 56s) |
| ⚠️ `ratio` é IGNORADO | pedi 1K/16:9 → voltou **1024×1024**. Cai no default 1:1 |
| ✅ **CONTORNO** | **`size:"1312x736"` (pixels explícitos), SEM `ratio`** → 16:9 preservado |
| Preserva | **estilo e composição** ✅ |
| NÃO preserva (1 ref) | ⚠️ **identidade** — rosto muda a cada imagem |
| Preserva (2+ refs) | ✅ **identidade** melhora muito |

### 5.1 ⚠️⚠️ 5 REFERÊNCIAS DESTROEM A IMAGEM — o teto útil é 2

Medido na mesma cena (a alavanca), variando só o número de refs:

| Refs | Resultado |
|---|---|
| 0 (text2img) | cena OK, personagem deriva |
| **2 (1 Althmann + 1 Bento)** | ✅ **MELHOR** — ação preservada, personagens certos, sem artefato |
| 3 (só Althmann) | ✅ OK |
| **5** | ❌ **QUEBRA TOTAL** |

Com 5 refs: **confete colorido** (pixels aleatórios) sobre a imagem toda **e o prompt é
ignorado** — a saída vira uma cópia da composição das âncoras. 16 imagens da série saíram
assim: todas "esquilo e castor lado a lado num galho", sem a cena pedida (sem alavanca,
sem castor preso, sem ponte, sem pôr do sol).

❌ **Hipótese derrubada:** achei que o problema fosse **misturar dois personagens**. Não é —
1+1 foi o **melhor** resultado da grade. O problema é **saturação por quantidade**.

⚠️ **A API retornou HTTP 200 nas 16.** O status não sabe se a imagem presta.

### 5.2 Model sheet: derivar, nunca gerar em paralelo
❌ **Erro conceitual meu:** gerar N vistas com text2img **não** dá um model sheet — dá N
personagens diferentes, porque text2img não trava identidade (ovo e galinha).
✅ **Certo:** 1 âncora-mãe em text2img → **as outras vistas derivadas dela via img2img**.
Resultado: mesmo indivíduo em todas as vistas.
⚠️ Preço: a "composition preservation" puxa as vistas para a pose da mãe — o model sheet
sai com **pouca variedade de ângulo** (pedi perfil, veio três-quartos).

**Endpoints:** `/v1/images/edits` **existe** (não documentado no guia; 500 pedindo `image`) ✅ · `/v1/images/variations` → **501 não implementado**.

---

## 6. ✅ Comportamento do modelo (matriz de 23 imagens, 10 estilos × 2 assuntos + 3 avatares)

**O modelo é forte em AMBIENTE e fraco em PERSONAGEM.** Isso define o que ele serve para fazer.

| Ponto forte | Ponto fraco |
|---|---|
| Cena/paisagem — 10/10 estilos limpos | **Cauda dupla: 9/10** em pose frontal |
| Texto curto e grande | Identidade inconsistente entre imagens |
| Avatar humano fotorrealista (mãos fora de quadro) | "castanha"→muffin em 7/10 estilos |
| 10/10 estilos legíveis e distintos | Contagem e ação complexa |

### ✅ O bug da cauda dupla — e a cura
- Aparece em **pose frontal simétrica**, em qualquer estilo (realista, Pixar, aquarela, anime, óleo, flat, isométrico).
- **Não** aparece em perfil / três-quartos.
- **Não** é efeito de idioma (EN frontal também deu 2 caudas).
- ✅ **CURA: pedir explicitamente "ONE SINGLE tail, exactly one tail only"** — funcionou em PT **e** EN.

### ✅ Contaminação por descritor de estilo
Estilo com carga de personagem injeta gente/bicho em prompt de cenário:
- `pixar: "...pelagem detalhada, olhos expressivos"` → esquilo **de cabelo humano** numa ponte.
- `aquarela: "ilustração de livro infantil"` → uma **menina** correndo na ponte.
- Os 8 estilos com descritor puramente estético → paisagem limpa.

### Estilos — qual escolher
- **Pixar:** o mais obediente. Único que distingue **castor de esquilo** e acerta a castanha. Denso demais em cena (domar a paleta).
- **Aquarela:** lindo em close; ⚠️ na cena a dois, **o castor vira um bicho genérico** — fatal para a história.
- **Realista / cinematográfico:** excelentes em paisagem.
- **Futurista:** ⚠️ sequestra atributos — esquilo ruivo virou **prateado**.

---

## 6.5 ✅ VÍDEO — `agnes-video-v2.0` (testado e FUNCIONA)

Assíncrono: `POST /v1/videos` → `GET /agnesapi?video_id=<ID>` (`queued`→`in_progress`→`completed`|`failed`).

```bash
curl -X POST https://apihub.agnes-ai.com/v1/videos \
  -H "Authorization: Bearer $AGNES_API_KEY" -H "Content-Type: application/json" \
  -d '{"model":"agnes-video-v2.0",
       "prompt":"Smooth cinematic transition between the keyframes...",
       "num_frames":81, "frame_rate":24, "width":1312, "height":736,
       "extra_body":{"image":["<URL_A>","<URL_B>"], "mode":"keyframes"}}'
```

| Fato | Status |
|---|---|
| `mode: "keyframes"` com par A→B | ✅ **funciona** — interpola de verdade, personagem consistente |
| **3+ keyframes** | ✅ aceito (a doc diz "multiple keyframes") |
| ✅ **`seed`** | **existe e é aceito** — o modelo de IMAGEM não tem! |
| ✅ **`negative_prompt`** | existe (imagem não tem) ❓ não testado |
| Tempo | **19–50s** por clipe de 3,4s. Muito mais rápido que imagem 4K |
| Duração | `seconds = num_frames / frame_rate`; `num_frames ≤ 441`, regra **8n+1**; fps 1–60 |
| Custo | US$ 0/segundo |
| ⚠️ **RATE LIMIT REAL** | **`allows 5 requests per 1 minute(s)`** → HTTP 429. **O ÚNICO limite real observado o dia todo.** ⚠️ O guia diz "vídeo 1 req/min" — **a API diz 5**. Exige throttle |
| ❌ **URLs** | a doc diz que keyframes exigem **URL pública** — **MENTIRA**: **data URI base64 funciona** (testado, payload 3,2 MB) |
| ⚠️ Teto de duração | 441 frames = **18,4s @24fps**. Narração mais longa que isso não cabe num clipe |

⚠️ **A resposta da criação MENTE sobre o tamanho.** Ela devolveu `size=1312x736`, mas o MP4
real veio **1280×704** (ffprobe). 1280/704 = 1.82, **não é 16:9 exato** (1.78) → leve
distorção. A doc avisa que normaliza para o tier mais próximo (480p/720p/1080p).
**Sempre conferir o arquivo, não o JSON.**

✅ Duração confere: 81 frames @ 24fps = 3,375s exatos. Codec h264.

**Encaixe com a série:** as imagens A/B de cada cena são keyframe1/keyframe2 naturais →
13 clipes ≈ 65s de filme. Requer as URLs públicas das cenas (o gerador agora salva em
`urls_cenas.json`).

---

### 6.6 ⚠️ Engasgo intermitente da API (medido 2026-08-25/26/27 e 2026-09-13)
De vez em quando um `POST /v1/videos` ou o `GET` de status **conecta e fica mais de 120 s sem devolver um byte**. No Python isso sobe como `TimeoutError: The read operation timed out` (timeout de LEITURA — só o de conexão vira `URLError`); numa das vezes foi `_ssl.c: The handshake operation timed out`.
- **Não é** 503 `video_queue_full`, não é 429, não é task perdida: no mesmo minuto `GET /v1/models` responde 200 em 0,6–0,8 s, e a API volta sozinha em poucos minutos.
- Não aparece na doc oficial nem em mensagem de erro da API — só se vê no cliente.
- **Tratamento:** repetir com backoff (2/4/8 s), e se persistir esperar 30–60 s e insistir dentro do teto de polling. Uma task já aceita continua processando no servidor; abortar joga o vídeo fora.
- Custou 5 clipes no musicavideo (MVD#103, #119, #122, #174, #175) até virar retry em `providers/base.py` + `providers/agnes.py` (2026-09-13).

## 7. Modelos disponíveis (`GET /v1/models`) ✅
`agnes-image-2.1-flash` · `agnes-image-2.0-flash` * · `agnes-2.0-flash` · `agnes-1.5-flash` * · `agnes-video-v2.0`
\* fora do guia INEMA. ❓ nenhum dos dois testado.

Stack real (vazou em mensagens de erro): **gateway litellm** sobre **one-api**; modelo interno `agnes-t2i-general-model`.
O modelo de texto **recusa** revelar system prompt — não insistimos.

---

## 8. Riscos 📘
- Sem SLA no free; cotas e políticas podem mudar.
- No free **seus dados podem treinar os modelos**, salvo opt-out ❓ (não achei onde se faz).
- Usar como camada gratuita de alto volume, com fallback. Nunca como fornecedor único crítico.

---

## 9. Em aberto
- [ ] Multi-âncora (3–5 vistas) melhora identidade? ← **o mais promissor**
- [ ] `size:"1312x736"` explícito contorna o `ratio` ignorado no img2img?
- [ ] Array aceita 6+? (refazer com timeout > 300s)
- [ ] Por que "castanha"→muffin depende do estilo?
- [ ] 2K: tempo e taxa de falha (só temos 1K e 4K)
- [ ] Taxa real de 503 no 4K (amostra atual: 2)
- [ ] Validade da URL em `platform-outputs.agnes-ai.space`
- [ ] `/v1/images/edits`: payload correto
- [ ] `agnes-image-2.0-flash` é melhor/pior?
- [ ] Contexto real do `agnes-2.0-flash`: 512K ou 256K?
- [ ] Onde fica o opt-out de treinamento

---

## 10. Registro de testes

### 2026-07-17 — sessão 1 (~70 chamadas)
- Probe de params/endpoints; amostras de texto; aquarela vs Pixar; img2img; escada de referências; matriz 10 estilos × 2 assuntos + 3 avatares; PT vs EN (6 pares).
- Artefatos: `~/projetos/output/agnes-nei/{probe,estilos,img2img,qualidade,pt-en}/` + `CONTACT-{A,B,avatares,PT-EN}.png`
- Taxa de 503 na matriz: **12 retries / 23 imagens (~34%)**, 100% recuperadas no retry.
