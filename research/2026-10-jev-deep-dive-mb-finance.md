# Jev (TypeSafe AI): pesquisa completa, do básico ao avançado, aplicada à MB Finance

> Pesquisa feita em 19/10/2026. Foco: **agentes de voz da MB Finance**, hub B2B de crédito PJ, conta PJ e adquirência.
>
> **Como esta pesquisa foi feita.** Além de artigos, benchmarks e reviews, **li o código-fonte dos SDKs oficiais e das integrações**: `typesafe-sdk` 0.7.5 (Python), `@typesafe-ai/sdk` 0.7.0 (TypeScript), `pipecat-ai` 1.12.0, `langchain-typesafe` 0.0.1a3, o decorator `jev` 0.3.0 e o crate Rust `typesafe-jev` 0.2.0. O schema da API (seção 3) vem dos modelos Pydantic **gerados automaticamente a partir do `openapi.json` oficial** (`https://api.typesafe.ai/openapi.json`), por isso é a fonte mais confiável deste documento.
>
> **Legenda de confiança:** 🟢 fonte primária (código/spec) ou várias fontes independentes concordando · 🟡 uma fonte só ou fonte do próprio fornecedor · 🔴 fontes conflitantes.

---

## 0. TL;DR

- **O que é:** o Jev é um modelo que **não gera texto**. Você envia um `state` (o contexto) e um conjunto de **perguntas tipadas**. Ele devolve **probabilidades**: sim/não (`noul`), escolha entre opções (`choice`) ou nota numa escala (`score`). Todas as perguntas são respondidas **em paralelo, numa única chamada**. 🟢
- **Por que interessa para voz:** a latência independente fica em **~0,15–0,35 s (p50–p95)**, bem estável. O custo é **US$ 0,042 por milhão de tokens de entrada, com saída grátis**. Isso permite rodar **uma bateria de checagens a cada turno** da ligação sem estourar o orçamento de latência nem de custo. 🟢/🟡
- **Já tem integração nativa no Pipecat 1.12:** `JevClassifier`, `VoicemailDetector` (detecção de caixa postal em ligação ativa) e juiz de evals. Também há middlewares no LangChain para **roteamento de modelo** e **bloqueio de tool calls arriscadas**. 🟢
- **A ressalva principal é a acurácia, que é mista.** Em perguntas "difíceis" ou mal formuladas, o Jev perde para LLMs médios: 62,6% contra 81,3% do Haiku em phishing, por exemplo. **Quando a decisão é quebrada em perguntas atômicas e a lógica fica no código, sobe para 95%.** Os números do próprio fornecedor medem concordância com LLMs, não verdade de campo. Há indícios de **queda fora do inglês**, e **não existe benchmark em PT-BR**. 🟡
- **Veredito para a MB:** vale **pilotar** o Jev como **camada de decisão rápida** do agente de voz: detecção de caixa postal, roteamento, opt-out, pedido de humano, guardrails, QA de 100% das ligações e juiz de evals. **Não deve** ser usado para decidir crédito, extrair dados ou como única barreira de compliance. Antes de produção é preciso um **golden set em PT-BR** e **due diligence de LGPD** (ZDR, DPA, transferência internacional).

---

## 1. O básico

### 1.1 "System One" contra "System Two"

A TypeSafe usa a metáfora de Kahneman:

| | **System Two (LLMs: GPT, Claude, Gemini)** | **System One (Jev)** |
|---|---|---|
| Saída | Texto livre, token a token | Probabilidades sobre opções **definidas por você** |
| Latência típica | Segundos (ou dezenas, com raciocínio) | Centenas de ms, **estável** |
| Custo | Entrada e saída cobradas | Só entrada (US$ 0,042/M) |
| Formato | Precisa de parsing ou structured output (pode falhar) | **Sempre** no schema |
| Bom para | Gerar, explicar, extrair, planejar | **Julgar, classificar, rotear, pontuar** |

### 1.2 Quem fez

- **TypeSafe AI** (San Francisco), fundada por **Diogo Almeida**, coautor do InstructGPT e que trabalhou com RLHF e ChatGPT na OpenAI até 2024, junto com **Erik Gafni** e **Sasha Sheng**. 🟢
- Saiu do stealth com **~US$ 40M liderados pela DCVC**. 🟢
- **Lançamento do Jev: 15/09/2026**, primeiro modelo público da empresa. Começou em early access com waitlist; depois várias fontes passaram a tratá-lo como aberto ao público. 🟢
- Adoção rápida: **Vercel, Cloudflare, LangChain e Langfuse integraram em 3 dias**. Hoje há também Pipecat, LiteLLM, Pydantic AI, Promptfoo, Zuplo, OpenRouter e AIMLAPI. 🟢

### 1.3 O problema que ele resolve

Hoje, a maioria dos agentes usa **LLMs grandes como classificadores**: "responda em JSON qual é a intenção". Isso traz três problemas:
1. **Latência alta e variável**, o que é péssimo em voz, onde cada 100 ms conta.
2. **Saída frágil**: JSON quebrado, rótulo fora da lista, texto extra.
3. **Probabilidade inexistente ou mal calibrada**, então não dá para definir thresholds de negócio.

O Jev ataca os três: é rápido, **sempre** respeita o schema e devolve uma **distribuição de probabilidade** que a TypeSafe diz ser calibrada. Essa calibração é contestada (seção 5).

---

## 2. Conceitos fundamentais

### 2.1 `state`
É o **contexto** sobre o qual todas as perguntas da chamada são feitas. Pode ser:
- **texto** (`"Cliente: quero antecipar meus recebíveis..."`);
- **objeto JSON** (recomendado; veja 6.3), como `{"transcript": [...], "lead": {...}, "ultima_fala_cliente": "..."}`;
- **lista**;
- **imagem** (experimental, seção 3.6).

### 2.2 Os três tipos de pergunta (mais um experimental)

| Tipo | Pergunta | Resposta | Uso típico |
|---|---|---|---|
| **`noul`** (sim/não) | `instructions` e, opcionalmente, `criteria: {true, false}` descrevendo o que conta como sim e como não | `noul`: P(sim) de 0 a 1 | Gatilhos: opt-out, pedido de humano, caixa postal, injection |
| **`choice`** | `instructions` e `criteria: {rótulo: descrição \| null}` (até 255 opções) | `choice` (rótulo mais provável), `confidence`, `probabilities` por rótulo (soma ≈ 1) | Roteamento de intenção, tabulação, departamento |
| **`score`** | `instructions` e `criteria: [nível0, nível1, ...]` (lista ordenada) | `score` (média ponderada, pode ser fracionária), `confidence`, `legend`, `probabilities` por nível | Frustração, urgência, aderência a produto, qualidade |
| `box` *(experimental)* | `instructions` descrevendo o que localizar numa imagem | Coordenadas normalizadas e `confidence` | Fora do escopo de voz |

> **Limites** 🔴: o decorator `jev` e o Pipecat citam **255 opções por choice**. Para score, o crate Rust fala em **2 a 10 níveis**, e o decorator aceita até 256. Confirme na documentação oficial; na prática, use **3 a 5 níveis**.

### 2.3 Paralelismo
"Todas as perguntas de uma requisição são avaliadas contra o mesmo state **em paralelo**, então uma requisição com centenas de perguntas leva aproximadamente o mesmo tempo que uma com uma pergunta" (README do crate Rust; a Vals AI relata o mesmo de 1 a 32 perguntas). 🟢/🟡
**Implicação:** em voz, **agrupe todas as checagens do turno em uma única chamada**.

### 2.4 Calibração
A TypeSafe afirma que as probabilidades são **calibradas**: das respostas `noul` = 0,9, cerca de 90% são "sim". Ela atribui isso ao método de treino (RLCD). **Avaliações independentes contestam** (seção 5). **Trate a calibração como algo a medir nos seus dados, não como garantia.**

---

## 3. A API em detalhe (fonte: OpenAPI oficial, via SDK)

### 3.1 Endpoints 🟢

| Método | Path | Descrição |
|---|---|---|
| `POST` | `https://api.typesafe.ai/v1/systemone` | Avalia perguntas sobre um state |
| `GET` | `https://api.typesafe.ai/v1/models` | Lista modelos e aliases disponíveis |

- **Auth:** `Authorization: Bearer <TYPESAFE_API_KEY>`. A chave é obtida no `console.typesafe.ai`.
- **Rastreio:** toda resposta traz o header `x-typesafe-request-id`. **Logue esse valor**, porque é o que o suporte vai pedir.
- Variáveis de ambiente padrão dos SDKs: `TYPESAFE_API_KEY`, `TYPESAFE_BASE_URL`, `TYPESAFE_DEFAULT_MODEL`, `TYPESAFE_LOG_LEVEL`.

### 3.2 Request 🟢

```json
POST /v1/systemone
{
  "model": "jev-latest",
  "state": { "ultima_fala_cliente": "Pode tirar meu número da lista, não quero mais ligação.",
             "produto_ofertado": "antecipação de recebíveis" },
  "questions": {
    "opt_out": {
      "type": "noul",
      "instructions": "O cliente pediu para não ser mais contatado?",
      "criteria": {
        "true":  "Pede explicitamente para parar ligações, remover o número ou não ser contatado.",
        "false": "Apenas recusa a oferta atual ou pede para ligar em outro momento."
      }
    },
    "intencao": {
      "type": "choice",
      "instructions": "O que o cliente quer neste momento?",
      "criteria": {
        "seguir_oferta": "Quer continuar ouvindo ou contratar o produto ofertado",
        "outro_produto": "Pergunta sobre outro produto (capital de giro, conta PJ, maquininha)",
        "falar_humano": "Pede para falar com uma pessoa",
        "encerrar": "Quer encerrar a ligação",
        "nao_aplicavel": "Nenhuma das anteriores ou fala sem conteúdo"
      }
    },
    "frustracao": {
      "type": "score",
      "instructions": "Quão irritado o cliente está?",
      "criteria": ["Calmo", "Impaciente", "Irritado", "Muito irritado ou ofensivo"]
    }
  }
}
```

Campos:
- `model` (obrigatório na wire; os SDKs preenchem `jev-latest`). Também existem **`jev-preview`** (modelo com visão, para imagens) e **versões fixas** como **`jev-1.13.0`**, que é o default do Pipecat ("fixado para que os thresholds calibrados continuem valendo").
- `state`: string, objeto ou lista (obrigatório).
- `questions`: mapa `nome → pergunta`, com pelo menos uma.
- `assets` (opcional): imagens, veja 3.6.

### 3.3 Response 🟢

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "opt_out":   { "type": "noul",   "noul": 0.97 },
    "intencao":  { "type": "choice", "choice": "encerrar", "confidence": 0.81,
                   "probabilities": { "seguir_oferta": 0.02, "outro_produto": 0.01,
                                      "falar_humano": 0.03, "encerrar": 0.91, "nao_aplicavel": 0.03 } },
    "frustracao":{ "type": "score",  "score": 1.6, "confidence": 0.7,
                   "legend": { "0": "Calmo", "1": "Impaciente", "2": "Irritado", "3": "Muito irritado ou ofensivo" },
                   "probabilities": { "0": 0.05, "1": 0.35, "2": 0.55, "3": 0.05 } }
  },
  "usage": { "input_tokens": 312, "output_tokens": 9 }
}
```

- `model` na resposta mostra **qual versão concreta respondeu**, mesmo quando você mandou um alias. **Logue esse campo.**
- `output_tokens` aparece, mas a spec diz: *"Output tokens are currently free of charge"*.

### 3.4 Instruções e critérios estruturados 🟢
`instructions` e os valores de `criteria` aceitam **texto, objeto JSON ou lista**. O padrão recomendado pela documentação dos SDKs é **nomear campos do state entre crases** dentro da pergunta:

```json
"instructions": {
  "politica": "Não é permitido prometer aprovação de crédito antes da análise.",
  "question": "A `ultima_fala_bot` viola a `politica`?"
}
```

### 3.5 Erros, retry e transporte 🟢 (comportamento dos SDKs oficiais)

| HTTP | Exceção (Python) | Retry padrão? |
|---|---|---|
| 400 | `TypeSafeBadRequestError` | não |
| 401 / 403 | `TypeSafeAuthenticationError` / `TypeSafePermissionDeniedError` | não |
| 422 | `TypeSafeUnprocessableEntityError` (detalhe com `loc`, `msg`, `type`) | não |
| 429 | `TypeSafeRateLimitError` (`retry_after_ms`) | **sim** |
| 408, 5xx (inclusive 529) | `TypeSafeInternalServerError` | **sim** |
| timeout / conexão | `TypeSafeAPITimeoutError` / `TypeSafeAPIConnectionError` | **sim** |

- **Defaults do SDK Python:** timeout de **10 s** por operação; `RetryPolicy(max_retries=2, backoff 0,5 s → 5 s, jitter 25%, orçamento total de 30 s)`; respeita `Retry-After` e `retry-after-ms`.
  ⚠️ **Em voz, esses defaults são altos demais.** Configure **timeout de 0,6–0,8 s e `max_retries=0` ou 1** no caminho síncrono do turno (veja 7.4).
- Pool de conexões: 100 conexões em keep-alive de 30 s. O Pipecat usa **HTTP/2**, keep-alive de 240 s e `connect()` antecipado no `setup()` para **não pagar handshake TLS no primeiro turno**.
- Compressão **zstd** do corpo quando passa de 1 KB (Python 3.14 ou extra `[zstd]`), útil com transcripts longos.

### 3.6 Imagens (experimental) 🟢
É preciso o header **`TypeSafe-Danger-Zone-Images: true`** e o modelo `jev-preview`. Aceita PNG ou JPEG via multipart ou base64. O próprio nome do header deixa claro: **não use em produção**. Para a MB isso seria relevante no futuro para documentos de KYB, mas hoje não é o caminho.

### 3.7 Limites operacionais 🔴 (fontes secundárias, confirmar com a TypeSafe)
- **Contexto:** ~64k tokens por requisição, dos quais no máximo ~32k para `state` mais a maior pergunta.
- **Rate limit:** os valores divergem entre fontes (1.200 req/min, 80 req/s, 40 req/s). Negocie o limite por contrato.
- **Idioma:** entrada **só texto** (fora o experimental de imagem). **Não há lista oficial de idiomas suportados.**

---

## 4. SDKs e ecossistema (o que existe hoje)

| Pacote | Versão | O que oferece | Observação |
|---|---|---|---|
| **`typesafe-sdk`** (Python) | 0.7.5 | Cliente sync e async, `Noul`/`Choice`/`Score`/`Box`, `RetryPolicy`, `response_model` Pydantic customizado | Oficial. `pip install typesafe-sdk` (o pacote `typesafe-ai` é só um redirect) |
| **`@typesafe-ai/sdk`** (TS/JS) | 0.7.0 | `client.systemOne({state, questions})`, helpers `noul()`/`choice()`/`score()`, tipos inferidos | Oficial. **Node 22+** |
| **`jev`** (Python) | 0.3.0 | `@jev.fn`: uma função com retorno Pydantic vira uma consulta (`bool`→noul, `Literal`→choice, `int`/`float` com `ge`/`le`→score) | Comunidade. Python 3.14. Ótimo para QA pós-chamada |
| **`typesafe-jev`** (Rust) | 0.2.0 | Cliente tipado, retries, contagem de custo | **Não oficial** (vem do projeto jevgrep) |
| **Pipecat** | 1.12.0 | `JevClassifier`, `JevClient`, `VoicemailDetector`, juiz de evals | **A peça mais relevante para voz** (veja 4.1) |
| **`langchain-typesafe`** / `@langchain/typesafe` | 0.0.1a3 / 0.0.2 | `TypeSafeClassifier` (Runnable). Experimentais: **`ModelRouterMiddleware`** e **`AutoModeMiddleware`** | Alpha |
| LiteLLM, Pydantic AI, Promptfoo, Zuplo, Langfuse, Vercel, Cloudflare, OpenRouter | n/a | Pass-through, gateway, evals, observabilidade | Útil para centralizar chaves e logs |

### 4.1 Pipecat 1.12, em detalhe 🟢 (lido do código-fonte)

O Pipecat (BSD-2, da Daily) criou uma **abstração de classificador** (`BaseClassifier`) com duas implementações: `LLMClassifier` (qualquer LLM) e **`JevClassifier`**. Isso importa porque **evita lock-in**: dá para trocar Jev por LLM, ou por um classificador próprio, mudando uma linha.

- **API:** `await classifier.ask(state, {"nome": YesNoQuestion(...) | ChoiceQuestion(...) | ScoreQuestion(...)})`, ou os atalhos `yes_no()`, `choice()` e `score()`. Emite o evento `on_metrics` com latência e tokens. Em falha ou timeout lança `ClassifierError`, e **é obrigatório ter fallback**.
- **`JevClient`** compartilhável entre vários classificadores. Usa HTTP/2, pré-conecta no `setup()`, faz retry só em 429 e 529, e o **modelo fica fixado em `jev-1.13.0`**.
- **`VoicemailDetector(classifier=JevClassifier(...), decision_timeout=1.0, voicemail_response_delay=2.0)`**: colocado depois do STT, consulta o classificador **em background a cada transcrição** e segura a fala do bot (`TTSGate`) até decidir entre `conversation` e `voicemail`. ⚠️ A decisão sai depois de `decision_timeout` (1 s por padrão) de silêncio do outro lado, mesmo que o Jev responda em 100 ms. **É esse timeout que domina a latência, não o modelo.**
- **Juiz de evals** (`pipecat/evals/judge.py`): roda cenários scriptados e pergunta ao classificador se a resposta do bot atende ao critério. As opções são `yes`, `no`, `continue` e `not_applicable`. Com Jev, "um cenário consegue distinguir um veredito certo de um caso duvidoso", e o `explainer` (um LLM) só é acionado quando a confiança fica abaixo de `explain_below` (0,75).
- A estratégia de turno `ExternalUserTurnCompletionStopStrategy` aceita "um classificador dedicado de fim de turno". Não encontrei benchmark do Jev nesse papel, e eu **não** o usaria para turn-taking (veja 7.2).

### 4.2 LangChain: dois middlewares relevantes 🟢 (lidos do código)
- **`ModelRouterMiddleware`**: uma `Choice` decide **qual LLM** atende cada requisição, por exemplo um modelo barato para dúvida simples e um forte para negociação. Isso corta custo do backend agêntico.
- **`AutoModeMiddleware`**: antes de **executar cada tool call**, uma `Noul` pergunta "executar isto é arriscado ou não autorizado pelo usuário?". O prompt padrão trata todo conteúdo do state como **dado, não instrução**, e bloqueia acima de 0,5. Para a MB, serve como uma **segunda camada** antes de tools de escrita (alterar cadastro, enviar proposta). **Não substitui** a política determinística de autorização.

---

## 5. Desempenho real: o que as evidências dizem

### 5.1 Latência 🟢 (várias fontes concordam)

| Fonte | Medição |
|---|---|
| TypeSafe | 70–500 ms ponta a ponta |
| Telemetria OpenRouter (01/10) | **p50 0,21 s · p95 0,34 s · p99 0,75 s** |
| Respan (via OpenRouter) | mediana 0,19 s (OpenAI Decisions: 0,15 s) |
| Benchmark independente (Medium) | **5–6x mais rápido que LLMs na mediana** (não 40–200x); **p95 < 500 ms** contra 3–3,5 s dos LLMs |
| Classmethod (Japão) | 0,25–0,32 s via NeMo Switchyard; 0,64–0,69 s via API direta (diferença atribuída ao cliente) |
| Vals AI | ~0,1 s, de 1 a 32 perguntas |
| Docs Pipecat | "tipicamente cerca de um décimo de segundo" |

**Conclusão:** espere **200–350 ms** a partir do Brasil, por causa da distância até os EUA. O ganho real está menos na média e mais na **cauda estável**, que é o que importa em voz.

### 5.2 Acurácia 🟡 (resultados mistos)

| Teste | Resultado |
|---|---|
| Dashboard da TypeSafe (711 casos) | Jev 67,8% contra 74,1% do melhor comparador. **O gabarito é a média de GPT-6 Astra e Claude Fable 5.1**, ou seja, mede concordância com LLMs, não verdade |
| Red Hat, guardrails de content safety | **1º lugar, 86,20%** (Qwen 85,47%; Granite Guardian 80,27%) |
| LegalBench (via review independente) | **Último de 12** em acurácia bruta e balanceada |
| Phishing, 2.000 e-mails, **uma pergunta** | **62,6%** contra 81,3% do Claude Haiku |
| Phishing, **decomposto em 5 perguntas**, pesos ajustados em metade dos dados | **95,0%** na outra metade |
| Processamento de faturas | 61,8% contra 79,1% de modelos de fronteira |
| Respan, contra OpenAI Decisions API | Jev **89,2% contra 71,4%** e **97,6% contra 90,5%**; Brier 0,117 contra 0,296 (Jev melhor). Outras fontes dizem que não houve comparação pareada |
| DEV Community, 8 dias de testes | "No nível de LLMs de preço médio, atrás da fronteira" |

### 5.3 Calibração 🔴
- A TypeSafe afirma calibração, mas **não publica ECE, Brier, reliability plot nem log loss** por dataset.
- No teste de spam independente: **ECE do Jev 0,053, contra 0,006 do modelo de fronteira**; até o LLM barato se saiu melhor.
- No comparativo com o OpenAI Decisions, o Jev teve o **melhor Brier** (0,117).
- **Conclusão:** calibre os thresholds **nos seus dados**, com reliability plot e, se preciso, Platt ou isotonic por cima das probabilidades.

### 5.4 Idioma 🟡 (o maior risco para a MB)
- Relatos de queda fora do inglês: **russo no XNLI de 88,3% para 77,3%, com ECE triplicando**; **espanhol perde 3–6 pontos**.
- **Nenhum dado público em português.** Como PT-BR é próximo do espanhol, uma estimativa razoável é **perda de alguns pontos e pior calibração**. Isso precisa ser medido.
- Mitigação a testar: escrever `instructions` e `criteria` **em inglês** e mandar o `state` em PT-BR, e comparar com tudo em português.

### 5.5 Limitações estruturais 🟢
- **"Não alucina" significa apenas "não sai do schema".** O Jev pode responder `departamento = cobranca` com confiança alta quando o certo era `suporte`.
- Fraco com **dupla negação**, **condições com escopo** e **múltiplos saltos de referência**. **Não use como calculadora.**
- **Sensível à formulação** da pergunta.
- Não extrai valores (CNPJ, nome, valor desejado). Para isso, use LLM, regex ou o STT com formatação.

---

## 6. Como escrever perguntas que funcionam (a prática)

Estas regras vêm da documentação dos SDKs, do juiz de evals do Pipecat e dos testes independentes:

1. **Uma pergunta por fato atômico; a lógica E/OU fica no código.**
   ❌ `"O cliente quer crédito e tem faturamento acima de 50 mil e não está negativado?"`
   ✅ três perguntas (`quer_credito`, `faixa_faturamento`, `mencionou_restricao`) e `if` no código.
   *Foi isso que levou phishing de 62,6% para 95%.*
2. **Toda resposta possível precisa ser uma opção.** Inclua `nao_aplicavel`, `incerto` ou `continue`. Não escreva "se não se aplicar, responda X" nas instruções: segundo os docs do Pipecat, *"um classificador segue uma instrução para todo critério, mesmo onde ela não cabe"*.
3. **Descreva os critérios**, principalmente a fronteira entre opções parecidas, como "recusar a oferta" contra "pedir opt-out".
4. **Use state estruturado e referencie campos com crases.** Separe `historico` de `ultima_fala_cliente` e de `ultima_fala_bot`, para o modelo saber **o que** julgar.
5. **Evite negações** ("O cliente NÃO deixou de..."). Reescreva na forma positiva.
6. **Para score, use 3–5 níveis com descrições concretas**, e não "1 a 10".
7. **Use thresholds de negócio por pergunta**, sem ficar preso a 0,5. Por exemplo, opt-out com P ≥ 0,35, porque o custo do falso negativo é alto (LGPD, reclamação, Procon).
8. **Fixe a versão do modelo** (`jev-1.13.0` ou a vigente) e **versione as perguntas em git**. Trocar de versão exige reavaliar o golden set.
9. **Para decisões compostas, combine várias perguntas num modelo leve.** Use regressão logística sobre as probabilidades do Jev, treinada com rótulos humanos. Foi esse "pesos ajustados" que levou ao 95%.
10. **Use banda de incerteza:** quando `confidence` estiver baixa ou P ficar entre 0,4 e 0,6, mande para LLM, revisão humana ou um caminho conservador.

---

## 7. Arquitetura para os agentes de voz da MB

### 7.1 Onde o Jev entra

```
                ┌──────────────────────────────────────────────────────────────┐
 Telefonia ───► │ STT streaming (PT-BR) ─► Turn detection (Smart Turn/Flux/VAD) │
 (SIP/WebRTC)   └───────────────┬──────────────────────────────────────────────┘
                                │ transcrição final do turno
                ┌───────────────▼───────────────────────┐
                │  JEV GATE (1 chamada, N perguntas)     │  ~200–350 ms, em paralelo
                │  opt_out · falar_humano · intencao ·   │  com o início do LLM
                │  injection · frustracao · vulneravel   │
                └───────┬───────────────────────┬────────┘
             decisão determinística             │ (sem gatilho)
             em código (thresholds)             ▼
   ┌───────────────────┴─────┐      ┌──────────────────────────────┐
   │ Ações imediatas:         │      │ LLM / backend agêntico       │
   │ • encerrar c/ script     │      │ (roteado por ModelRouter)    │
   │ • transferir p/ humano   │      │  └─ tool calls ─► AutoMode   │
   │ • registrar opt-out      │      │     (Jev) ─► Policy engine   │
   │ • trocar de fluxo        │      │     determinístico ─► MCP    │
   └──────────────────────────┘      └──────────────┬───────────────┘
                                                    │ resposta do bot
                                     ┌──────────────▼───────────────┐
                                     │ (opcional) Jev compliance     │
                                     │ na fala do bot antes do TTS   │
                                     └──────────────┬───────────────┘
                                                    ▼
                                                  TTS
 Pós-chamada (assíncrono): Jev QA em 100% das ligações · tabulação · score de lead
 Offline: juiz de evals (Pipecat) · golden set PT-BR · monitor de drift
```

### 7.2 Orçamento de latência: onde ele cabe e onde não cabe

| Ponto da pipeline | Cabe? | Por quê |
|---|---|---|
| **Detecção de fim de turno** (a cada ~100 ms de áudio) | ❌ | Precisa de < 50 ms e roda local. Use Smart Turn (Pipecat, BSD), LiveKit Turn Detector ou Deepgram Flux |
| **Gate por turno**, depois da transcrição final | ✅ | 200–350 ms **em paralelo** com o primeiro token do LLM. Se disparar um gatilho, cancela a resposta do LLM |
| **Compliance da fala do bot antes do TTS** | ⚠️ | Soma ~250 ms ao tempo até o primeiro áudio. Use só em fluxos críticos (oferta de taxa, condições) ou por sentença em streaming |
| **Caixa postal e URA em outbound** | ✅ | Já existe no Pipecat. A latência é dominada pelo `decision_timeout` |
| **Pós-chamada e QA** | ✅✅ | Sem restrição de latência; custo desprezível |

**Padrão recomendado (gate especulativo):**
1. A transcrição final chega e você dispara **ao mesmo tempo** o `Jev.ask(...)` e a geração do LLM.
2. O Jev volta em ~250 ms. Se algum gatilho passar do threshold, **cancela** a geração (o Pipecat tem interrupção nativa) e executa a ação determinística.
3. Se o Jev **estourar o timeout (~700 ms) ou falhar**, segue com o LLM e marca o turno para auditoria. Em perguntas de segurança, como opt-out, o fallback é um **regex ou palavra-chave conservador**.

### 7.3 Infra e operação

| Tema | Recomendação |
|---|---|
| **Cliente** | Um `JevClient` por processo, compartilhado, com HTTP/2, pré-conectado no startup e keep-alive longo |
| **Timeout e retry** | Caminho síncrono: timeout de 0,6–0,8 s e **0–1 retry**. Pós-chamada: defaults do SDK (10 s, 2 retries) |
| **Fallback** | `LLMClassifier` (abstração do Pipecat), classificador local ou regra conservadora. **Nunca** deixar uma falha do Jev bloquear a ligação |
| **Circuit breaker** | Com p95 > 600 ms ou taxa de erro > 2% numa janela de 1 min, desligar o gate síncrono e operar em modo degradado |
| **Gateway** | Passar por LiteLLM ou Zuplo internos para centralizar chave, rate limit, **redação de PII** e logs (veja 8) |
| **Observabilidade** | Logar por chamada `request_id`, `model` respondido, versão das perguntas, probabilidades, threshold aplicado, decisão e latência. Usar OTel com Langfuse (já integrado ao Jev) ou o `on_metrics` do Pipecat |
| **Versionamento** | Perguntas como código (YAML ou Pydantic) em git, com **golden set rodando em CI** a cada mudança de pergunta ou de modelo |
| **Rate limit** | Cada turno faz uma chamada. Com 200 ligações simultâneas e um turno a cada ~5 s, são ~40 req/s, na faixa dos limites citados. **Negociar limite enterprise** |
| **Drift** | Monitorar a distribuição de `choice` e P(noul) por dia. Mudança brusca indica nova versão do modelo, mudança de script ou mudança de público |

### 7.4 Custo estimado
Supondo um state de ~1.500 tokens por turno (janela de transcript, lead e perguntas) e ~20 turnos do cliente por ligação, são **~30k tokens por ligação, ou ~US$ 0,0013**.
Com **100 mil ligações por mês**, o gate custa **~US$ 130/mês**. O QA pós-chamada (transcript inteiro, ~8k tokens) adiciona ~US$ 35/mês.
➡️ **O custo é irrelevante.** A decisão depende de **acurácia, latência e compliance**, não de preço.

---

## 8. LGPD, compliance e due diligence (crítico para fintech)

| Tema | Situação | Ação |
|---|---|---|
| **Treino com dados** | A TypeSafe diz que **não treina** com requests e respostas de clientes 🟡 | Exigir isso em contrato (DPA) |
| **Retenção** | **ZDR (zero data retention) só para enterprise, sob pedido**, não é o padrão 🟡 | **Exigir ZDR** antes de mandar transcripts reais |
| **SOC 2** | 🔴 Uma fonte cita SOC 2 Type II (2026); outra diz que a documentação pública não menciona | Pedir o relatório |
| **Região** | Nenhuma opção regional encontrada; API nos EUA 🟡 | **Transferência internacional (LGPD, arts. 33 e seguintes)**: avaliar com o DPO a base legal e as cláusulas-padrão |
| **Minimização** | | Mandar **só o necessário**: a última fala e uma janela curta. **Redigir CPF, CNPJ de pessoa física, dados bancários e endereço** no gateway antes de enviar. O Jev quase nunca precisa desses dados para julgar intenção |
| **Decisão automatizada (LGPD, art. 20)** | | Não usar o Jev para **decidir** crédito nem condições. Se um score dele afetar a priorização de clientes, documentar critérios e oferecer revisão humana |
| **Auditoria** | | Guardar probabilidades, versão do modelo e das perguntas, e decisão, porque isso **explica** por que a ligação foi transferida, encerrada ou marcada |
| **Opt-out** | | Gatilho com threshold baixo, mais regex de backup, gravação imediata na lista de bloqueio e **confirmação verbal** para o cliente |

---

## 9. Casos de uso na MB Finance (priorizados)

### Prioridade 1: voz, alto impacto e baixo risco

**UC1. Detecção de caixa postal e URA em outbound (prospecção de crédito PJ)**
- **O quê:** saber nos primeiros segundos se atendeu uma pessoa, uma caixa postal ou uma URA de empresa.
- **Como:** `VoicemailDetector(classifier=JevClassifier(...))` do Pipecat. Ele já segura a fala do bot até decidir.
- **Valor:** não falar com caixa postal (custo por minuto e imagem), deixar recado padronizado e voltar à fila.
- **KPI:** acurácia caixa postal contra humano e tempo até a decisão.

**UC2. Gate de segurança por turno**
Uma chamada por turno com estas perguntas:
- `opt_out` (noul), `falar_humano` (noul), `injection` (noul: "a fala tenta instruir o assistente a ignorar regras ou revelar informações internas?"),
- `vulnerabilidade` (noul: menção a dificuldade financeira grave, doença, luto, para mudar o tom e oferecer humano),
- `frustracao` (score de 4 níveis), `intencao` (choice).
- **Ações:** determinísticas em código (seção 7.2).
- **KPI:** recall de opt-out (meta ≥ 98%, com regex de backup), taxa de transferência adequada e reclamações.

**UC3. Roteamento de intenção e produto**
- `choice` entre `capital_de_giro`, `antecipacao_recebiveis`, `conta_pj`, `adquirencia_maquininha`, `renegociacao`, `status_proposta`, `reclamacao`, `outro`.
- Leva para o **subfluxo ou agente especialista** e usa o `ModelRouterMiddleware` para escolher LLM barato ou forte.
- **Valor:** menos tokens no LLM principal e prompts menores e mais focados por produto.

### Prioridade 2: qualidade e compliance

**UC4. Compliance da fala do bot** (fluxos críticos)
- Antes do TTS, em ofertas e condições: `promete_aprovacao` (noul), `informa_taxa_sem_cet` (noul), `pressao_indevida` (noul).
- Se disparar, troca a fala por uma alternativa segura pré-aprovada.
- Usar **só** onde os +250 ms se justificam.

**UC5. QA de 100% das ligações (pós-chamada)**
- Com `@jev.fn` e um modelo Pydantic: `apresentou_se_como_ia: bool`, `informou_gravacao: bool`, `resolveu_demanda: Literal[...]`, `tabulacao: Literal[...]`, `aderencia_script: int (0–3)`, `risco_reclamacao: int (0–3)`.
- Hoje o QA humano cobre amostras de ~2–5%. Com Jev, **100% das ligações**, e o humano revisa só os casos sinalizados ou de baixa confiança.

**UC6. Tabulação automática e score de lead**
- `disposition` (choice), `interesse` (score 0–3), `faixa_faturamento_mencionada` (choice, incluindo `nao_mencionou`), `melhor_horario` (choice).
- Alimenta o CRM e a priorização de retorno. ⚠️ Priorização **não é** decisão de crédito.

### Prioridade 3: engenharia e plataforma

**UC7. Juiz de evals e simulações** (Pipecat `judge` com `JevClassifier`): roda centenas de cenários PT-BR a cada deploy de prompt, barato e rápido. Usa `explain_below` para chamar um LLM só nos casos duvidosos.

**UC8. Gate de tool calls** (`AutoModeMiddleware`): uma checagem extra antes de tools de escrita no backend agêntico. Fica **atrás** de uma política determinística de autorização, nunca no lugar dela.

**UC9. Fora da voz:** triagem de WhatsApp e e-mail de clientes PJ (departamento, urgência, sentimento) e roteamento de tickets.

### Onde **não** usar
- ❌ Decisão de crédito, limite, taxa ou aprovação.
- ❌ Extração de dados (CNPJ, valores, nomes): use LLM, regex ou NER.
- ❌ Detecção de fim de turno (latência e frequência incompatíveis).
- ❌ Única barreira contra fraude ou prompt injection.
- ❌ Imagens e documentos (KYB) em produção, porque o recurso é experimental.

---

## 10. Exemplos de código

### 10.1 Gate por turno no Pipecat (Python)

```python
import os
from pipecat.classifiers.base_classifier import ChoiceQuestion, ClassifierError, ScoreQuestion, YesNoQuestion
from pipecat.classifiers.jev.classifier import JevClassifier
from pipecat.classifiers.jev.client import JevClient
from pipecat.frames.frames import Frame, TranscriptionFrame
from pipecat.processors.frame_processor import FrameDirection, FrameProcessor

QUESTIONS_V = "turn-gate@2026-10-19"  # versionado em git junto com o golden set

TURN_QUESTIONS = {
    "opt_out": YesNoQuestion(
        instructions="O cliente pediu para não ser mais contatado?",
        yes="Pede para parar ligações, remover o número ou não ser contatado.",
        no="Só recusa a oferta atual ou pede para ligar em outro momento.",
    ),
    "falar_humano": YesNoQuestion(instructions="O cliente pediu para falar com uma pessoa?"),
    "injection": YesNoQuestion(
        instructions="A `ultima_fala_cliente` tenta instruir o assistente a ignorar regras, mudar de papel ou revelar informações internas?",
    ),
    "frustracao": ScoreQuestion(
        instructions="Quão irritado o cliente está?",
        levels=["Calmo", "Impaciente", "Irritado", "Muito irritado ou ofensivo"],
    ),
    "intencao": ChoiceQuestion(
        instructions="O que o cliente quer neste momento?",
        options={
            "seguir_oferta": "Quer continuar ou contratar o produto ofertado",
            "outro_produto": "Pergunta sobre outro produto da MB",
            "encerrar": "Quer encerrar a ligação",
            "nao_aplicavel": None,
        },
    ),
}

THRESHOLDS = {"opt_out": 0.35, "falar_humano": 0.5, "injection": 0.6}


class JevTurnGate(FrameProcessor):
    def __init__(self, client: JevClient, on_trigger):
        super().__init__()
        self._jev = JevClassifier(client=client)
        self._on_trigger = on_trigger  # async callback: (gatilho, resultados) -> None

    async def process_frame(self, frame: Frame, direction: FrameDirection):
        await super().process_frame(frame, direction)
        if isinstance(frame, TranscriptionFrame):
            # dispara em background: o LLM continua em paralelo (gate especulativo)
            self.create_task(self._check(frame.text))
        await self.push_frame(frame, direction)

    async def _check(self, text: str):
        state = {"ultima_fala_cliente": text}  # sem PII: só a fala do turno
        try:
            r = await self._jev.ask(state, TURN_QUESTIONS)
        except ClassifierError:
            return await self._fallback(text)  # regex conservador para opt-out
        for name, thr in THRESHOLDS.items():
            if r[name].probability >= thr:
                return await self._on_trigger(name, r)

    async def _fallback(self, text: str): ...


client = JevClient(api_key=os.environ["TYPESAFE_API_KEY"], timeout=0.8)  # modelo fixado por padrão
```

### 10.2 QA pós-chamada com `@jev.fn`

```python
from typing import Literal
from pydantic import BaseModel, Field
import jev

class QALigacao(BaseModel):
    apresentou_se_como_ia: bool = Field(description="O assistente informou que é uma inteligência artificial?")
    informou_gravacao: bool = Field(description="O assistente informou que a ligação é gravada?")
    prometeu_aprovacao: bool = Field(description="O assistente prometeu aprovação de crédito antes da análise?")
    desfecho: Literal["proposta_enviada", "retorno_agendado", "sem_interesse", "opt_out", "transferido", "caixa_postal"]
    risco_reclamacao: int = Field(ge=0, le=3, description="Risco de o cliente registrar reclamação",
                                  json_schema_extra={"levels": ["nenhum", "baixo", "médio", "alto"]})

@jev.fn
def auditar(transcript: str) -> QALigacao:
    """Transcrição de uma ligação de prospecção de crédito PJ da MB Finance:

    {{ transcript }}
    """
    return auditar.state()
```

### 10.3 Chamada HTTP direta (qualquer linguagem)

```bash
curl -s https://api.typesafe.ai/v1/systemone \
  -H "Authorization: Bearer $TYPESAFE_API_KEY" -H "Content-Type: application/json" \
  -d '{"model":"jev-latest","state":{"ultima_fala_cliente":"me tira dessa lista"},
       "questions":{"opt_out":{"type":"noul","instructions":"O cliente pediu para não ser mais contatado?"}}}'
```

---

## 11. Plano de implementação (6–8 semanas)

| Fase | Duração | Entregas | Critério de saída |
|---|---|---|---|
| **0. Due diligence** | 1 sem (em paralelo) | DPA, **ZDR**, relatório SOC 2, base legal para transferência internacional, rate limit enterprise, SLA | Aprovação do DPO e de Segurança |
| **1. Golden set PT-BR** | 1–2 sem | 1.000–2.000 turnos reais **anonimizados**, rotulados por 2 pessoas (opt-out, humano, intenção, frustração, injection), com concordância entre anotadores medida | Kappa ≥ 0,7 |
| **2. Avaliação offline** | 1 sem | Comparar **Jev (PT e EN nas instruções)**, **OpenAI Decisions API**, **LLM pequeno como classificador** e **classificador fine-tuned PT-BR** (BERTimbau ou ModernBERT multilíngue). Medir recall/precisão por pergunta, **ECE e reliability**, p50/p95 a partir de São Paulo e custo | Jev ≥ alternativa em recall de opt-out e humano, com p95 < 500 ms |
| **3. Shadow mode** | 2 sem | Gate rodando em produção **sem agir**, só logando. Comparar com o comportamento atual e com revisão humana por amostragem | Falsos positivos de transferência e encerramento abaixo do limite acordado |
| **4. Produção gradual** | 2 sem | UC1 e UC2 ativos em 10%, 50% e 100% das ligações, com circuit breaker e fallback. Depois UC5 (QA) | KPIs da seção 9 estáveis |
| **5. Expansão** | contínuo | UC3, UC4, UC6, UC7 e UC8, com reavaliação a cada nova versão do modelo | |

---

## 12. Alternativas e mitigação de lock-in

| Opção | Prós | Contras |
|---|---|---|
| **Jev** | Mais barato (US$ 0,042/M), latência estável, ecossistema forte (Pipecat, LangChain) | Startup nova, acurácia mista, PT-BR não medido, sem região BR/UE |
| **OpenAI Decisions API** (beta pública desde 06/10/2026, `gpt-6-luna`, `/v1/decisions`) | Mesmos 3 tipos de pergunta (predicate, choice, score); aceita imagens; respostas mais consistentes entre execuções; mediana ~0,15 s | US$ 0,10/M; **beta**; um teste a mostra menos precisa e pior calibrada que o Jev |
| **Clef** (Clef-flash, US$ 0,09/M) | Outra opção do mesmo nicho | Pouca informação pública |
| **Classificador próprio** (BERTimbau ou ModernBERT fine-tuned) | Roda **na sua infra** (LGPD resolvida), ~10–30 ms, treinado em PT-BR com seus dados | Exige rótulos, MLOps, retreino e uma pergunta "fixa" por modelo (sem pergunta nova em runtime) |
| **LLM pequeno como juiz** | Flexível, sem fornecedor novo | Mais lento e caro, com saída menos confiável |

**Mitigação:** programe contra a **interface de classificador** (`BaseClassifier` do Pipecat ou uma interface própria `ask(state, questions)`), não contra o SDK do Jev. Mantenha o golden set e as perguntas em formato neutro. Trocar de fornecedor vira configuração.

**Estratégia sugerida:** **Jev para perguntas novas e em evolução** (sem treino, muda o texto e pronto). Quando uma pergunta estabilizar e tiver volume, como opt-out, **migrar para um classificador próprio em PT-BR** e manter o Jev como segunda opinião.

---

## 13. Checklist de perguntas para a TypeSafe

1. Existe **ZDR** no nosso plano? E retenção de logs para debug (quantos dias, onde)?
2. **Região de processamento?** Há roadmap para Brasil ou UE? E sub-processadores?
3. **SOC 2 Type II** (relatório), pentest, ISO 27001?
4. **Qual foi o desempenho em português?** Há dados de avaliação ou recomendações (instruções em EN ou PT)?
5. **Rate limit** contratável e **SLA** de latência e disponibilidade? Status page?
6. **Política de versões:** quanto tempo uma versão fixada (ex.: `jev-1.13.0`) fica disponível? Há aviso de depreciação?
7. Existem **métricas de calibração** publicadas (ECE, Brier) por domínio?
8. Existe opção **on-prem ou VPC**?
9. Limites exatos: contexto, níveis de score, tamanho de state.

---

## 14. Fontes

**Primárias (código e spec, lidos nesta pesquisa)**
- OpenAPI via SDK Python: `typesafe-sdk` 0.7.5, [PyPI](https://pypi.org/project/typesafe-sdk/), modelos gerados de `https://api.typesafe.ai/openapi.json`
- [`@typesafe-ai/sdk` 0.7.0 (npm)](https://www.npmjs.com/package/@typesafe-ai/sdk) · [`jev` 0.3.0 (PyPI)](https://pypi.org/project/jev/) · [`typesafe-jev` 0.2.0 (crates.io)](https://crates.io/crates/typesafe-jev) / [docs.rs](https://docs.rs/typesafe-jev)
- [`pipecat-ai` 1.12.0](https://pypi.org/project/pipecat-ai/): `pipecat/classifiers/jev/*`, `extensions/voicemail/voicemail_detector.py`, `evals/judge.py` · [Docs Pipecat: Jev classifier](https://docs.pipecat.ai/api-reference/server/classifiers/jev.md) · [Classifiers](https://docs.pipecat.ai/pipecat/learn/classifiers) · [Voicemail](https://docs.pipecat.ai/pipecat/fundamentals/voicemail) · [Release 1.12.0](https://newreleases.io/project/github/pipecat-ai/pipecat/release/v1.12.0)
- [`langchain-typesafe`](https://pypi.org/project/langchain-typesafe/) (classifier, ModelRouterMiddleware, AutoModeMiddleware) · [Docs LangChain](https://docs.langchain.com/oss/python/integrations/providers/typesafe)
- Documentação oficial: [docs.typesafe.ai](https://docs.typesafe.ai/) · [API reference](https://docs.typesafe.ai/api) (não acessível deste ambiente; referenciada pelos SDKs)

**Lançamento e empresa**
- [The Rundown: TypeSafe launches Jev](https://www.therundown.ai/news/typesafe-jev-ai-decisions-software) · [InfoQ](https://www.infoq.com/news/2026/10/typesafe-ai-jev-released/) · [Winzheng (US$ 40M)](https://www.winzheng.com/en/article/typesafe-ai-jev-system-one-model-launch) · [Startup Fortune (Vercel)](https://startupfortune.com/typesafe-ais-decision-model-jev-becomes-vercels-fastest-adopted-launch/) · [Diogo Almeida no AI Engineer](https://ai.engineer/speakers/diogo-almeida)

**Avaliações independentes e críticas**
- [Independent Benchmark (Medium)](https://medium.com/@pravvich/typesafes-jev-beyond-the-hype-an-independent-benchmark-8bdc1c99d000) · [Vals AI: Independent Evaluation of Jev](https://www.vals.ai/blogs/independent-evaluation-of-jev) · [DEV: 8 dias de testes](https://dev.to/gde/jev-after-eight-days-of-independent-tests-level-with-mid-price-llms-behind-the-frontier-1kln) · [KDnuggets](https://www.kdnuggets.com/what-everyone-is-getting-wrong-about-typesafe-ais-jev) · [Beri: 62,6% contra 95% com decomposição](https://www.beri.net/article/typesafe-jev-typed-decision-model-calibration-decomposition-shadow-eval) · [XenoSpectrum: decomposição](https://xenospectrum.com/jev-typesafe-bert-classifier-decomposition/) · [OrcaRouter: audits](https://www.orcarouter.ai/blog/jev-typesafe-system-one-what-we-know) · [OrcaRouter: limitações](https://www.orcarouter.ai/he/blog/jev-limitations) · [Layer3: review](https://www.layer3labs.io/guides/jev-review)
- [The New Stack: Red Hat guardrail benchmark](https://thenewstack.io/red-hat-guardrail-benchmark/) · [Arize: guardrails em tempo real com Jev](https://arize.com/?p=32158) · [Classmethod: Jev + NeMo Switchyard](https://dev.classmethod.jp/en/articles/jev-support-for-nemo-switchyard/)

**Voz**
- [Evalgent: Jev para roteamento e escalonamento em voz](https://www.evalgent.com/blog/jev-voice-agent-routing-escalation) · [Evalgent: Decisions API contra Jev em voz](https://www.evalgent.com/blog/openai-decisions-api-vs-jev-voice-agents) · [Evalgent: detecção de caixa postal](https://www.evalgent.com/blog/voicemail-detection-livekit-pipecat) · [Deepgram: projetos open source com Jev](https://deepgram.com/learn/audio-decision-composition-five-open-source-jev-projects) · [LiveKit: turn detection](https://livekit.com/blog/turn-detection-voice-agents-vad-endpointing-model-based-detection)

**Segurança, preço e dados**
- [NeuralTrust: guia de segurança enterprise](https://neuraltrust.ai/blog/jev-typesafe-ai-security) · [Prophet Security](https://www.prophetsecurity.ai/blog/jev-security-operations) · [Appinventiv: casos e limites](https://appinventiv.com/blog/jev-usecases-and-adoption/) · [Jev ZDR e retenção](https://jevaiguide.com/faq/does-jev-train-on-your-data/) · [Hokai: rate limits](https://hokai.io/hub/models/jev) · [innFactory](https://innfactory.ai/en/ai-models/typesafe-jev/)

**Concorrentes**
- [Unite.AI: OpenAI Decisions API](https://www.unite.ai/openai-releases-decisions-api-in-public-beta-powered-by-gpt-6-luna/) · [MarkTechPost](https://www.marktechpost.com/2026/10/09/openai-decisions-api-hits-public-beta-with-10x-faster-typed-answers/) · [Respan: Decisions API contra Jev](https://www.respan.ai/articles/openai-decisions-api-vs-jev) · [CometAPI](https://www.cometapi.com/openai-decisions-api-vs-jev-pricing-speed-usage/) · [Developers Digest: Decisions, Jev e Clef](https://www.developersdigest.tech/blog/openai-decisions-api-vs-jev-vs-clef-2026) · [Classmethod: Decisions overview](https://dev.classmethod.jp/en/articles/openai-decisions-overview/) · [Hora de Codar (PT-BR)](https://horadecodar.com.br/?p=47329)
