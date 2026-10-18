# Inovações em AI (jul–out/2026) aplicáveis à arquitetura da MB Finance

> Pesquisa feita em 18/10/2026. Foco em **infraestrutura e arquitetura de projetos de AI**, sem produtos B2C de topo de funil, com atenção especial a **agentes de voz** e ao **Jev (TypeSafe AI)**.
>
> **Premissa:** a MB Finance é tratada aqui como o hub B2B do Rio de Janeiro que conecta empresas a crédito PJ, abertura de conta PJ e adquirência. Se o contexto for outro, as recomendações das seções 7 e 8 mudam um pouco, mas as tendências continuam valendo.
>
> **Sobre as fontes:** boa parte vem de anúncios de fornecedores, blogs e agregadores. Vários sites primários (arXiv, InfoQ, Towards AI) estavam bloqueados por rede neste ambiente, então algumas informações vêm de resumos de busca. Cada afirmação leva um nível de confiança: 🟢 fonte primária ou várias fontes concordando; 🟡 uma fonte só ou fonte de fornecedor; 🔴 dado conflitante.

---

## 1. Resumo executivo

1. **Agentes de voz agora usam um frontend de conversa separado de um backend que executa.** O modelo de voz full-duplex cuida da conversa e delega tool calls e raciocínio para um agente de texto assíncrono. GPT-Live-1 (OpenAI), Qwen-Audio-Agent e um paper de setembro (arXiv 2609.19334) usam essa arquitetura. Para fintech, essa separação é também o ponto onde ficam os controles de compliance.
2. **Surgiu uma camada de decisão tipada separada do LLM.** O **Jev** (TypeSafe AI, 15/09/2026) não gera texto: recebe um estado e perguntas tipadas (sim/não, escolha, score) e devolve probabilidades. É bom para roteamento, triagem e guardrails a cada turno, rápido e barato. A acurácia ainda não foi comprovada de forma independente.
3. **Agentes em produção dependem de três coisas:** *durable execution* (Temporal, Restate, Inngest), *context management* como disciplina de arquitetura e observabilidade com OpenTelemetry GenAI.
4. **O MCP ficou stateless** (spec 2026-07-28) e teve a autenticação endurecida. Isso facilita escalar servidores MCP internos atrás de load balancer comum.
5. **Na segurança contra prompt injection, a solução está saindo do modelo** e indo para políticas determinísticas que mediam as ações do agente (CaMeL, FIDES, Progent). Para uma fintech, isso já deveria ser requisito.
6. **Regulação:** o PL 2338 (Marco Legal da IA) só volta a ser votado depois das eleições de outubro de 2026. A LGPD (art. 20, revisão de decisões automatizadas) já vale hoje e afeta qualquer triagem de crédito automatizada.

---

## 2. Agentes de voz: o que mudou

### 2.1 Lançamentos recentes de modelos speech-to-speech

| Data | Modelo | Destaque | Confiança |
|---|---|---|---|
| 06/07/2026 | OpenAI gpt-realtime-2.1 | Raciocínio dentro do loop de áudio | 🔴 versões conflitantes entre fontes |
| 10/09/2026 | **OpenAI GPT-Live-1** (API) | Full-duplex (ouve e fala ao mesmo tempo); delega para um backend escolhido pelo dev; ~US$ 0,05/min só a camada de voz | 🟢 data / 🟡 preço |
| 15/09/2026 | Google Gemini 3.8 Live | Novo modelo padrão da Live API para baixa latência | 🟡 |
| 14/10/2026 | **xAI Grok Voice 3.0** | US$ 0,08/min; segundo a própria xAI, 74% no EVA-Bench (líder) | 🟡 números da própria xAI |
| set/2026 | NemotronLabs VoiceChat (NVIDIA, arXiv 2609.21967) | Full-duplex **open** com tool calling | 🟡 |
| set/2026 | Qwen-Audio-Agent (arXiv 2609.25195) | Divisão foreground/background, também open | 🟡 |
| Build 2026 | Azure Voice Live (GA) + Azure-Realtime (preview) | Opção para quem já está em Azure | 🟡 |

### 2.2 A mudança de arquitetura: frontend duplex com backend delegado

O paper **"A frontend-backend architecture for tool calls in full-duplex speech models"** (arXiv 2609.19334, set/2026) descreve bem o padrão:

- Um frontend duplex de fala aprende a emitir um **token de delegação** e envia o transcript em streaming para um **LLM de texto no backend**, que faz as tool calls.
- O resultado volta por um mecanismo de *prefill-and-repeat* e é falado por TTS em streaming.
- Resultados: **92–97% de recall em tool calls** e **81,2% de acerto ao recusar chamadas irrelevantes**, mantendo turn-taking e interrupções.
- Motivação, de fonte secundária 🟡: em tarefas de atendimento com dados reais, modelos duplex comerciais completam **31–51%**, contra **~85%** de agentes de texto. Ou seja, **o modelo de voz não deveria ser o agente principal.**

O **GPT-Live-1** segue o mesmo desenho: a sessão de voz cria um *delegation ID*, manda contexto ao backend e recebe o resultado via `session.commentary.append`. O Pipecat já tem um modo *client delegation*, em que **você fornece o backend** (`BackendLLMWorker`). Isso evita lock-in no modelo de raciocínio.

> Um ponto citado com frequência: *"a camada de fala não deve mutar diretamente contas, reservas ou bancos de dados"*. Em fintech, toda escrita passa pelo backend, que tem política, idempotência e auditoria.

### 2.3 Arquitetura de referência sugerida para a MB Finance

```
 Telefonia/SIP / WebRTC
        │
 ┌──────▼──────────────────────────────────────────┐
 │  FRONTEND DE VOZ (tempo real, < 800 ms)          │
 │  • S2S full-duplex (GPT-Live-1 / Gemini Live /    │
 │    Grok Voice) OU cascata STT→LLM→TTS            │
 │  • Turn detection (Pipecat Smart Turn, BSD)      │
 │  • Só conversa: não executa nada sensível        │
 └──────┬───────────────────────────▲───────────────┘
        │ delegação (intenção +     │ resultado
        │ transcript + estado)      │ verbalizável
 ┌──────▼───────────────────────────┴───────────────┐
 │  CAMADA DE DECISÃO TIPADA (a cada turno, ~ms)     │
 │  • Jev ou classificador próprio:                  │
 │    intenção? urgência? risco/fraude? opt-out?     │
 │    tentativa de injection? escalar p/ humano?     │
 └──────┬───────────────────────────────────────────┘
        │
 ┌──────▼───────────────────────────────────────────┐
 │  BACKEND AGÊNTICO (texto, assíncrono)             │
 │  • LLM forte (Claude / GPT / open)                │
 │  • Durable execution (Temporal/Restate)           │
 │  • Policy engine determinístico (estilo CaMeL)    │
 │  • Tools via MCP (stateless) → core de crédito,   │
 │    KYC/KYB, CRM, bureaus, abertura de conta PJ    │
 └──────┬───────────────────────────────────────────┘
        │
  Observabilidade OTel GenAI · Evals (EVA-Bench-like,
  Coval/Cekura) · Trilhas de auditoria · LGPD
```

**Por que esse desenho funciona para a MB Finance:**
- Dá para **trocar o provedor de voz** sem reescrever a lógica de negócio, o que importa porque surgem modelos novos a cada 4–6 semanas.
- O backend recebe as **mesmas garantias de um sistema financeiro**: idempotência, retries sem cobrar LLM em dobro e auditoria.
- A camada de decisão fica barata o suficiente para rodar **em todo turno**, e não só por amostragem.

### 2.4 S2S ou cascata?

- **S2S** ganha em latência e naturalidade em casos sensíveis a tempo, como atendimento inbound e qualificação.
- **Cascata** (STT, LLM e TTS em streaming) dá mais controle por turno: transcript auditável, troca de LLM e vocabulário de domínio. Custa mais latência e engenharia de turn-taking.
- **Recomendação:** para fluxos regulados, como oferta de crédito, confirmação de dados ou cobrança, **cascata ou S2S com backend delegado e transcript persistido**.

### 2.5 STT e PT-BR: atenção

Todos os benchmarks recentes de STT encontrados (Pipecat, ago/2026; Soniox; Coval; AssemblyAI) são **em inglês**. Exemplos: NVIDIA Nemotron 3.0 ASR com mediana de 221 ms e 1,95% de WER; Deepgram nova-3 com 247 ms e 1,62%. **Não encontrei benchmark público de PT-BR.**
**Ação:** montar um *golden set* próprio com áudios reais (sotaques, ruído de telefonia, termos como "capital de giro", "antecipação de recebíveis", CNPJ ditado) e medir WER e P95 de latência por fornecedor.

### 2.6 Avaliação de agentes de voz

- **EVA-Bench** (ServiceNow, arXiv 2605.13841, open source): conversas bot-a-bot em áudio, métricas **EVA-A** (acurácia) e **EVA-X** (experiência), 213 cenários e perturbações de sotaque e ruído, com pass@k e **pass^k** (confiabilidade). Achado principal: **nenhum dos 12 sistemas passou de 0,5 em acurácia e experiência ao mesmo tempo**, e o pico de desempenho fica longe do desempenho confiável. A Data 2.0 aumentou os cenários em 4x. 🟢
  → **Recomendação:** usar a metodologia (pass^k, perturbação de ruído) para criar uma suíte interna com cenários de crédito PJ.
- **Plataformas de simulação** 🟡: Coval (regressão em CI/CD, Series A de US$ 28M), Cekura (pré-produção e monitoramento em produção), Hamming, Roark. O paper "Testing the Testers" (2026) indica que **a acurácia dessas plataformas ainda é baixa** (~43–49 pontos), então elas não substituem revisão humana amostral.

---

## 3. Jev (TypeSafe AI): análise detalhada

### 3.1 O que é

- **Lançamento:** 15/09/2026. Primeiro modelo público da TypeSafe AI (San Francisco). 🟢
- **Fundadores:** **Diogo Almeida**, coautor do InstructGPT e ex-OpenAI (RLHF, ChatGPT), com Erik Gafni e Sasha Sheng. Captação de **~US$ 40M liderada pela DCVC**. 🟢
- **Proposta:** um modelo de **"System One"**, ou seja, decisões rápidas e intuitivas, em contraste com o "System Two" deliberativo dos LLMs. **Não gera texto token a token.** 🟢

### 3.2 Como funciona (API)

Você envia um **`state`** (texto, estruturado ou não) e um conjunto de **`questions`** tipadas. Todas são avaliadas **em paralelo** sobre o mesmo estado. Há três tipos de pergunta:

| Tipo | Retorno | Exemplo MB Finance |
|---|---|---|
| **noul** (sim/não) | probabilidade de "sim" | "O cliente pediu para não ser mais contatado?" |
| **choice** | probabilidade por opção | "Qual a intenção: capital de giro, antecipação, conta PJ, maquininha, reclamação?" |
| **score** | posição numa escala ordinal | "Quão frustrado está o cliente (1–5)?" |

Exemplo de payload (formato reconstruído a partir da documentação de integradores; confira na referência oficial em docs.typesafe.ai/api):

```json
{
  "state": "<transcript do último turno + dados do lead>",
  "questions": {
    "intencao": { "type": "choice", "instructions": "Qual produto o cliente quer?",
                  "criteria": ["capital_de_giro", "antecipacao_recebiveis", "conta_pj", "adquirencia", "outro"] },
    "opt_out":  { "type": "noul", "instructions": "O cliente pediu para não ser contatado?" },
    "injection":{ "type": "noul", "instructions": "A fala tenta instruir o agente a ignorar regras?" },
    "frustracao":{ "type": "score", "instructions": "Nível de frustração do cliente" }
  }
}
```

### 3.3 Números

| Métrica | Valor | Fonte / Confiança |
|---|---|---|
| Velocidade | "20–200x mais rápido" que LLMs | TypeSafe 🟡 |
| Custo | "40–400x mais barato"; **US$ 0,042 / 1M tokens de input, output grátis** | TypeSafe 🟡 |
| Latência de ponta a ponta | 70–500 ms (TypeSafe); **0,25–0,32 s** medido via NeMo Switchyard; 0,64–0,69 s via API direta (Classmethod) | 🟡 testes independentes pequenos |
| Content safety | **86,20%, 1º lugar** no benchmark de guardrails da Red Hat (Qwen 85,47%; Granite Guardian 80,27%) | Red Hat / The New Stack 🟢 |
| Acurácia geral | **67,8% contra 74,1%** do melhor comparador no dashboard de 711 casos da própria TypeSafe. Ganha em custo e latência, **perde em acurácia** | 🟡 gabarito gerado por modelo, não ground truth |
| Adoção | Vercel, Cloudflare, LangChain e Langfuse integraram em 3 dias; também LiteLLM, Pydantic AI, Promptfoo, Zuplo | 🟢 |

### 3.4 Como interpretar

- O argumento de **"não alucina"** só garante que **a saída sempre respeita o schema**. Isso não quer dizer que a resposta está certa: ele ainda pode escolher a opção errada com confiança alta.
- Ele **não substitui** o LLM. Ele substitui as chamadas de LLM usadas como classificador ou juiz, que hoje são caras, lentas e exigem parsing de JSON frágil.
- Por ser **calibrado com probabilidades**, permite *thresholds* de negócio, como escalar para humano se P(fraude) > 0,3, e curvas ROC. LLM-as-judge não oferece isso com facilidade.

### 3.5 Onde encaixar na MB Finance

| Caso | Valor | Risco / cuidado |
|---|---|---|
| **Roteamento de intenção a cada turno do agente de voz** | Latência baixa o bastante para rodar dentro do turno; elimina prompt de roteamento no LLM | Medir acurácia em PT-BR |
| **Guardrails em tempo real** (injection, opt-out, linguagem abusiva, pedido de humano) | Barato o bastante para rodar em 100% dos turnos | Combinar com política determinística (seção 5) |
| **Detecção de obrigações de compliance** (cliente pediu revisão, contestação ou cancelamento) | Gatilhos auditáveis para workflows | Logar probabilidades e versão do modelo |
| **Triagem e priorização de leads PJ** (score ordinal de aderência a produto) | Pré-filtro barato antes do analista | **Não** usar como decisão de crédito sem explicabilidade e sem revisão humana (LGPD art. 20) |
| **Avaliação online de qualidade** (o agente seguiu o script? deu informação incorreta de taxa?) | Score em 100% das ligações em vez de 10–20% de amostra | Calibrar contra rótulos humanos |

### 3.6 Plano de piloto (2–3 semanas)

1. Separar **500–1.000 turnos reais**, anonimizados, de ligações ou chats e rotular intenção, opt-out e escalonamento por humanos.
2. Comparar **Jev**, **LLM pequeno como juiz** e **classificador fine-tuned** (por exemplo, ModernBERT ou Granite Guardian) em acurácia, P95 de latência e custo por 1k turnos.
3. Escolher o *threshold* pela curva de precisão e recall de cada pergunta.
4. Verificar **residência e retenção de dados** e DPA com a TypeSafe. Transcript de cliente é dado pessoal (LGPD) e, em parte, sigilo bancário. Se o contrato não fechar, usar o classificador próprio.
5. Limitação: no lançamento, o acesso era por **waitlist/early access**. Confirmar se já está disponível em GA ou por meio de gateways como AIMLAPI ou LiteLLM.

---

## 4. Infraestrutura de agentes

### 4.1 MCP spec 2026-07-28 (estável) 🟢
- **Core stateless:** o rastreamento de sessão saiu do protocolo. Versão, identidade do cliente e capabilities vão em metadados **por requisição**. Na prática, um servidor MCP vira um serviço HTTP comum, escalável horizontalmente e sem sticky sessions.
- **Auth:** credenciais presas ao issuer (SEP-2352). *Dynamic Client Registration* foi depreciado em favor de **Client ID Metadata Documents (CIMD)**. Há também *Enterprise-Managed Authorization*.
- **Depreciados:** Sampling e Roots. Tasks longas viraram extensão opcional.
- **Ação:** expor sistemas internos (core de crédito, CRM, bureaus) como **servidores MCP internos**, atrás do gateway e do IdP da empresa, e não com integrações ad-hoc por agente.

### 4.2 Agent Skills (SKILL.md) 🟢
- Padrão aberto publicado pela Anthropic (dez/2025), adotado por cerca de 40 ferramentas até jun/2026 (Claude Code, Codex, Copilot, Cursor, Gemini CLI etc.). A *progressive disclosure* mantém cada skill em ~100 tokens até ser usada.
- **Ação:** codificar procedimentos internos como skills versionadas em git (política de crédito, script regulatório, playbook de cobrança, checklist de KYB). Isso vale tanto para agentes de produção quanto para os desenvolvedores.

### 4.3 Durable execution 🟢/🟡
- Regras comuns: **o output do LLM é gravado uma vez no histórico e reaproveitado no replay**, sem nova chamada; o workflow precisa ser determinístico; toda tool com efeito colateral precisa ser idempotente.
- **Temporal** é o mais maduro (Nexus e multi-região GA em 2026, porém payloads grandes de LLM incham o histórico). **Restate** é mais leve e tem exactly-once nativo. **Inngest** é o caminho mais rápido em TypeScript, mas o custo por step cresce com retries.
- Dado de mercado 🟡: só **11–14% dos pilotos de agentes corporativos chegam a produção** (mar/2026).
- **Ação:** todo fluxo de agente que dure mais que uma ligação (onboarding PJ, coleta de documentos, follow-up de proposta) roda em um motor durável.

### 4.4 Context management como disciplina 🟡
- **"Agentic Context Management"** (arXiv 2607.21503, jul/2026): argumenta que a maioria das falhas de agente vem de **gestão ruim do que entra no contexto**, e não de raciocínio fraco. Propõe cinco primitivas: *Architecting, Ingesting, Scoping, Anticipating, Compacting*.
- Baseia-se no **ACE: Agentic Context Engineering** (arXiv 2510.04618, ICLR 2026), em que contextos evoluem como *playbooks*, com +10,6% em agentes e **+8,6% em tarefas de finanças**.
- Tendência no GitHub (out/2026, fonte fraca 🟡): **memória persistente** foi a categoria que mais cresceu. Microsoft (Copilot Studio, memória compartilhada) e DigitalOcean (Agent Data Services) lançaram camadas gerenciadas.
- **Ação:** para o agente de voz, ter um **"estado do cliente" estruturado e compacto** (lead, CNPJ, produto de interesse, objeções, consentimentos) que o backend mantém e injeta, em vez de reenviar transcripts longos.

### 4.5 Observabilidade 🟢
- As **OpenTelemetry GenAI Semantic Conventions** ganharam repositório dedicado (jun/2026) e ainda estão em status *Development*, então é preciso fixar versões. As convenções para multi-agente (tasks, actions, memória) estão em andamento. O Datadog já suporta nativamente.
- O OTel **não cobre avaliação**. Rodar LLM-as-judge ou Jev por cima. Guias sugerem 10–20% do tráfego para LLM-judge; com um classificador barato, dá para chegar a 100%.
- **Sanitizar PII antes de exportar spans** (CPF, CNPJ de pessoa física, dados bancários).
- Ferramentas: Langfuse (já integra Jev), Arize Phoenix, Braintrust, MLflow.
- Pesquisa a acompanhar: **NovaFabric** (arXiv 2609.12582), sobre evidências *tamper-evident* e reexecutáveis de execuções de agentes. Pode virar a base de trilha de auditoria para reguladores.

---

## 5. Segurança

- **A segurança está saindo do modelo** 🟢: a literatura de 2026 converge para uma **política determinística fora do LLM** que medeia cada ação (CaMeL, FIDES, Progent, RTBAS, FORGE), com ataques quase zerados no AgentDojo. Defesas por prompt ou fine-tuning **não têm garantia formal** e caem diante de ataques adaptativos.
  - **CaMeL:** um LLM privilegiado gera o plano a partir da instrução confiável, um LLM em quarentena lê os dados não confiáveis sem acesso a tools, e um interpretador rastreia a proveniência e aplica a política antes de cada tool call.
  - Para multi-agente, ver arXiv 2609.22949 (set/2026).
- **Exposição de infraestrutura** 🟢: a Censys (15/10/2026) mostra crescimento de **63% em 9 meses** de infra de AI exposta. Só o **n8n** tem cerca de 114 mil hosts expostos, guardando credenciais dos serviços conectados. Se a MB Finance usa n8n ou ferramenta similar, auditar já.
- **Governança** 🟢: um estudo da IBM (14/10/2026) aponta que 75% das empresas da EMEA esperam agentes com autonomia de decisão até 2027, mas só 37% têm governança preparada para isso.
- **Ação:** (a) política de autorização por tool, como "agente de voz pode consultar status de proposta mas não alterar dados bancários"; (b) confirmação fora de banda para ações sensíveis; (c) Jev ou classificador detectando injection na entrada; (d) inventário de credenciais em ferramentas de automação.

---

## 6. Modelos: open e locais

- **NVIDIA Nemotron 3.5** (out/2026) 🟡: **Super VL** de 120B (documentos, imagens, tools; fine-tunável com dados próprios) e modelos **Bolt** pequenos para rodar localmente ou na borda. Disponível em Hugging Face, OpenRouter e NIM.
- Relevância: **leitura de documentos de KYB** (contrato social, extratos, faturamento) com dados sensíveis sem sair da infraestrutura, e uma opção de *fallback* e *routing* para cortar custo em tarefas simples.
- Modelos de voz **open** com tool calling (NemotronLabs VoiceChat, Qwen-Audio-Agent) abrem caminho para um frontend de voz self-hosted, caso residência de dados ou custo por minuto vire problema.

---

## 7. Regulação (Brasil)

- **PL 2338/2023 (Marco Legal da IA):** aprovado no Senado em 10/12/2024 e parado na Câmara. O relator Aguinaldo Ribeiro disse em 24/08/2026 que a votação fica **para depois das eleições de outubro**. Depois volta ao Senado. 🟢
- **O que já vale hoje:** **LGPD art. 20**, que dá direito à revisão de decisões tomadas só com base em tratamento automatizado, inclusive perfil de crédito, além de transparência sobre os critérios. Também vale a regulação do Bacen/CMN sobre risco cibernético e contratação de nuvem para instituições reguladas, e as obrigações dos parceiros bancários da MB Finance, que costumam ser repassadas contratualmente.
- **Implicações de arquitetura:**
  - Logar **entradas, versão do modelo, probabilidades e decisão** de toda etapa automatizada. A camada de decisão tipada ajuda nisso.
  - Garantir **caminho de escalonamento humano** em fluxos de crédito.
  - Informar ao cliente que ele está falando com uma IA. O PL tende a exigir, e já é boa prática.

---

## 8. Roadmap sugerido

**0–30 dias**
- [ ] Golden set PT-BR (áudio e texto) e suíte de evals inspirada no EVA-Bench, com pass^k e ruído de telefonia.
- [ ] Piloto do Jev contra classificador próprio para intenção, opt-out e injection (seção 3.6).
- [ ] Auditoria de exposição: n8n e similares, credenciais em automações.
- [ ] Instrumentação OTel GenAI com sanitização de PII.

**30–60 dias**
- [ ] Refatorar o agente de voz para **frontend de conversa com backend delegado**. Testar GPT-Live-1 (client delegation via Pipecat) contra Gemini 3.8 Live, Grok Voice 3.0 e cascata.
- [ ] Expor sistemas internos como servidores MCP stateless atrás do IdP.
- [ ] Motor de durable execution para fluxos multi-sessão (onboarding PJ, documentos).

**60–90 dias**
- [ ] Policy engine determinístico (estilo CaMeL) entre o agente e as tools de escrita.
- [ ] Skills versionadas (SKILL.md) com políticas e scripts regulatórios.
- [ ] Avaliar Nemotron 3.5 ou outro modelo open para KYB documental on-prem.
- [ ] Simulação contínua em CI (Coval, Cekura ou a suíte interna) com revisão humana amostral.

---

## 9. Fontes

**Jev / TypeSafe**
- [The Rundown: TypeSafe launches Jev](https://www.therundown.ai/news/typesafe-jev-ai-decisions-software)
- [InfoQ: TypeSafe AI Releases Jev](https://www.infoq.com/news/2026/10/typesafe-ai-jev-released/)
- [Winzheng: Diogo Almeida launches Jev, US$ 40M](https://www.winzheng.com/en/article/typesafe-ai-jev-system-one-model-launch)
- [Startup Fortune: Vercel adoption](https://startupfortune.com/typesafe-ais-decision-model-jev-becomes-vercels-fastest-adopted-launch/)
- [The New Stack: Red Hat guardrail benchmark](https://thenewstack.io/red-hat-guardrail-benchmark/)
- [Arize: Real-time guardrails with Jev](https://arize.com/?p=32158)
- [Classmethod: Jev + NeMo Switchyard](https://dev.classmethod.jp/en/articles/jev-support-for-nemo-switchyard/)
- [OrcaRouter: what we know](https://www.orcarouter.ai/blog/jev-typesafe-system-one-what-we-know)
- [Towards AI: What is Jev](https://pub.towardsai.net/what-is-jev-ai-a-practical-guide-to-system-one-and-executable-decisions-c289a750a3ae)
- [AIMLAPI: Jev API](https://aimlapi.com/models/typesafe-jev-latest) · [docs.rs/typesafe-jev](https://docs.rs/typesafe-jev) · [Promptfoo provider](https://www.promptfoo.dev/docs/providers/typesafe/) · [LiteLLM](https://docs.litellm.ai/docs/pass_through/typesafe.md)
- [Diogo Almeida no AI Engineer](https://ai.engineer/speakers/diogo-almeida)

**Voz**
- [arXiv 2609.19334: Frontend-backend architecture for tool calls in full-duplex speech models](https://arxiv.org/pdf/2609.19334)
- [arXiv 2609.21967: NemotronLabs VoiceChat](https://arxiv.org/pdf/2609.21967) · [arXiv 2609.25195: Qwen-Audio-Agent](https://arxiv.org/pdf/2609.25195)
- [OpenAI: Introducing GPT-Live-1 in the API](https://openai.com/index/introducing-gpt-live-1-in-the-api/) · [OpenAI Live docs](https://developers.openai.com/api/docs/live) · [Pipecat OpenAI Live](https://docs.pipecat.ai/api-reference/server/services/s2s/openai-live)
- [Unite.AI: Grok Voice 3.0](https://www.unite.ai/xai-introduces-grok-voice-3-0-speech-to-speech-model-for-voice-agents/)
- [Azure Speech at Build 2026](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/azure-speech-at-build-2026-powering-voice-agents-with-real-time-and-life-like-ex/4524638)
- [AssemblyAI: best S2S API 2026](https://www.assemblyai.com/blog/best-speech-to-speech-voice-agent-api) · [Coval: Voice AI models 2026](https://www.coval.ai/blog/voice-ai-models-2026/)
- [EVA-Bench (arXiv 2605.13841)](https://arxiv.org/pdf/2605.13841) · [EVA-Bench leaderboard](https://www.benchlm.ai/voice-benchmarks/eva-bench)
- [Pipecat STT benchmarks](https://www.pipecat.ai/benchmarks/speech-to-text) · [Micdrop: open source voice frameworks](https://micdrop.dev/blog/open-source-voice-agent-frameworks) · [Evalgent: full-duplex](https://www.evalgent.com/blog/full-duplex-voice-agents)
- [Hackernoon: voice agent testing tools 2026](https://hackernoon.com/best-voice-agent-evaluation-and-testing-tools-in-2026) · [Coval vs Cekura](https://www.coval.ai/blog/coval-vs-cekura/)

**Infra, segurança e regulação**
- [MCP 2026-07-28 Specification](https://blog.modelcontextprotocol.io/posts/2026-07-28/) · [MCP Roadmap](https://blog.modelcontextprotocol.io/posts/mcp-roadmap/)
- [Agent Skills: SKILL.md adoption](https://www.noqta.tn/en/news/skill-md-open-standard-30-ai-coding-tools-adoption-2026)
- [Durable execution: Temporal / Inngest / Restate](https://particula.tech/blog/durable-execution-ai-agents-temporal-inngest-restate) · [Forkast: Temporal](https://forkast.news/temporal-wants-12-billion-to-keep-your-agents-from-crashing-mid-task/)
- [arXiv 2607.21503: Agentic Context Management](https://arxiv.org/abs/2607.21503) · [arXiv 2510.04618: ACE](https://arxiv.org/abs/2510.04618)
- [OTel GenAI conventions (TrueFoundry)](https://www.truefoundry.com/blog/opentelemetry-genai-semantic-conventions) · [Uptrace: OTel for AI](https://uptrace.dev/blog/opentelemetry-ai-systems) · [arXiv 2609.12582: NovaFabric](https://arxiv.org/pdf/2609.12582)
- [Design Patterns for Securing LLM Agents (arXiv 2506.08837)](https://arxiv.org/pdf/2506.08837) · [Adaptive Evaluation of Out-of-Band Defenses (arXiv 2606.26479)](https://arxiv.org/html/2606.26479v1) · [Multi-agent injection (arXiv 2609.22949)](https://arxiv.org/pdf/2609.22949)
- [Help Net Security: exposed AI infrastructure (Censys)](https://www.helpnetsecurity.com/2026/10/15/exposed-ai-infrastructure-report/) · [IBM EMEA AI governance study](https://newsroom.ibm.com/2026-10-14-ibm-study-eight-in-ten-emea-enterprises-escalate-ai-incidents-to-board-or-c-suite-as-ai-agents-test-governance)
- [TechRepublic: Nemotron 3.5](https://www.techrepublic.com/article/news-nvidia-nemotron-3-5-super-vl-bolt-ai-models/)
- [PL 2338 status (LCF monitor)](https://monitor.lcfconsulting.com.br/proposicoes/pl-2338-2023/) · [Bairo Gonzalez: where Brazil's AI framework stands](https://bairogonzalez.com/en/in-the-world/ia/pl-2338-brazil-ai-legal-framework-2026/)
- [MB Finance (Indeed)](https://br.indeed.com/cmp/Mb-Finance)
