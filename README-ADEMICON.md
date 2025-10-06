# Ademicon Chatbot — Flow (n8n) + Prompt (SPIN/LGPD)

Projeto oficial do **Consultor(a) Digital Ademicon** combinando **fluxo n8n** e **prompt estruturado** (SPIN + LGPD) para qualificação de leads e despacho ao **Apollo CRM**.

> Atualizado em: 2025-10-06  

---

## Visão Geral

- **Objetivo**: qualificar leads de consórcio (imóveis, veículos, serviços) via metodologia **SPIN**, capturar consentimento **LGPD** e encaminhar ao **especialista humano**.
- **Componentes**:
  - **Flow n8n** (`CHATBOT-ADEMICON-APOLLO.json`) — orquestração, validação, CRM.
  - **Prompt do agente** (`consultor_ademicon_prompt_v1.2.json`) — identidade, state machine, contrato de saída, validações e few-shots.
- **Canais**: web widget do n8n (Chat Trigger).

---

## Arquitetura

```
Usuário ↔ Chat Trigger (n8n)
   └─► 1. Extrair Input (Code)
        └─► 2. AI Agent Jonas (Agent + LLM + OutputParser)
             └─► 3. Processar AI (Code)  ──► 4. Enviar CRM? (IF)
                  │                            ├─► 5. Preparar Payload (Code)
                  │                            │    └─► 6. HTTP → Apollo CRM (POST)
                  │                            │         └─► 7. Tratar Resposta CRM (Code)
                  │                            │              └─► 8. Formatar Resposta (Code) → Chat
                  └────────────────────────────────────────────────────────► 8. Formatar Resposta (Code) → Chat
```

**Referência do flow**: o JSON inclui todos os nós acima, bem como as correções de parsing/anti-duplicação e o endpoint do Apollo. fileciteturn0file0

---

## Principais Recursos

- **State machine SPIN** com _gates_ por fase e fase inicial `S`.
- **Contrato JSON** com **JSON Schema**, limites de tamanho e **regra anti-duplicação** (`reply` ≠ `question`).
- **Normalização/validação**: email (regex), celular (regex, só dígitos), title case para nome/cidade.
- **Chips presets** por etapa (tipo de carta, status, urgência, consentimento).
- **Recuperação de erro**: mensagens simples para email/telefone inválidos e intenção desconhecida.
- **CRM Apollo** com tratamento de **duplicidade** e mensagens finais de sucesso/falha.
- **Memória** por janela (20 mensagens) para manter contexto no n8n.

---

## Estrutura de Pastas (sugerida)

```
.
├─ /flow/
│  └─ CHATBOT-ADEMICON-APOLLO.json
├─ /prompt/
│  └─ consultor_ademicon_prompt_v1.2.json
├─ /docs/
│  └─ json-contract.md
└─ README.md
```

---

## Prompt do Agente (resumo)

- **Identidade**: Consultor(a) Digital Ademicon — atendimento oficial, seguro e humano.
- **Tom**: profissional, claro, empático e transparente; **1 pergunta por turno**, **sem emojis/jargões**.
- **SPIN**: `S` (situação) → `P` (problema) → `I` (implicação) → `N` (necessidade) → `CLOSE` (LGPD + agendamento).
- **LGPD**: consentimento explícito antes do envio ao CRM.
- **Contrato de saída** (sempre):  
  ```json
  {
    "reply": "texto",
    "question": "uma pergunta objetiva",
    "chips": ["Opção1", "Opção2"],
    "conversationState": {
      "phase": "S|P|I|N|CLOSE",
      "nome": "", "email": "", "celular": "", "administradora": "",
      "status": "", "tipo_carta": "", "valor_atual_carta": "", "urgencia": "",
      "cidade": "", "observacoes": "", "consent_lgpd": false, "classificacao": "morno"
    },
    "shouldSendToCRM": false,
    "classificacao": "quente|morno|frio"
  }
  ```

---

## Flow n8n (detalhes dos nós)

1) **Chat Trigger**  
   - Mensagem inicial: “Oi! Eu sou o Jonas, consultor da Ademicon. Como posso te ajudar hoje?”  
   - `allowedOrigins`: configure o domínio de produção.

2) **1. Extrair Input (Code)**  
   - Normaliza entrada e inicializa `conversationState` canônico.
   - Log estruturado com timestamp (America/Sao_Paulo).

3) **2. AI Agent Jonas (Agent)**  
   - `systemMessage`: **prompt v1.2** embutido (state machine, schema, chips).  
   - Conectado ao **LLM** (OpenAI) e **Output Parser** (Structured).

4) **Memory (20msgs)**  
   - Mantém contexto curto da conversa.

5) **3. Processar AI (Code)**  
   - Parser robusto de JSON (remove fences, vírgulas finais, extrai bloco).  
   - **Anti-duplicação**: se `reply` contém `question`, zera `question`.  
   - Calcula `shouldSendToCRM` a partir de `conversationState` + `consent_lgpd`.

6) **4. Enviar CRM? (IF)**  
   - Bifurca para CRM se `shouldSendToCRM = true`.

7) **5. Preparar Payload (Code)**  
   - Normaliza celular e agrega observações (valor, cidade).  
   - Mapeia classificações auxiliares para Apollo.

8) **6. HTTP → Apollo CRM (POST)**  
   - **Boas práticas**: parametrizar URL/TOKEN por **variáveis de ambiente** (ver `.env`).

9) **7. Tratar Resposta CRM (Code)**  
   - Classifica `created`, `duplicate` ou `error`.

10) **8. Formatar Resposta (Code)**  
    - Produz mensagem final **sem concatenar** pergunta e resposta indevidamente.

---

## Variáveis de Ambiente

Crie/ajuste um `.env` na instância do n8n:

```bash
# OpenAI
OPENAI_API_KEY=sk-***

# Apollo CRM
APOLLO_CRM_URL=https://api.crmapollo.com.br/webhooks/leads/create.php
APOLLO_CRM_QUERY=url=VVL&user=Mw==&pipeline=0&token=***
APOLLO_CRM_TIMEOUT_MS=10000

# Chat
CHAT_ALLOWED_ORIGINS=https://seu-dominio.com
CHAT_WEBHOOK_ID=ghf-chatbot-webhook
```

> No nó **HTTP → Apollo CRM**, monte a URL como:  
> `${ $env.APOLLO_CRM_URL }?${ $env.APOLLO_CRM_QUERY }`

---

## Requisitos

- **n8n** ≥ 1.1x (self-hosted recomendado)  
- **OpenAI API Key**  
- Acesso ao **Apollo CRM** (webhook)  
- HTTPS/TLS no domínio público do chat

---

## Instalação & Uso

1. **Importe o flow** (`/flow/CHATBOT-ADEMICON-APOLLO.json`) no n8n.
2. **Configure credenciais** do OpenAI no n8n.
3. **Parametrize** o nó HTTP com variáveis de ambiente (evite tokens expostos).
4. **Ative o workflow** e **habilite o Chat Trigger**.
5. **Teste** pelo widget do n8n ou pelo endpoint de chat.

---

## Testes Rápidos

- **Saudação**: “oi” → pergunta por **nome**.  
- **Venda de carta**: “quero vender minha carta” → pergunta por **tipo** com chips.  
- **Consentimento LGPD**: dispara somente quando dados-chave estiverem válidos.  
- **Duplicidade CRM**: retorna mensagem dedicada sem erro fatal.

---

## Boas Práticas de Produção

- **Segurança**: CORS/CSP, rate limit, TLS, JWT para endpoints privados.
- **Observabilidade**: logs estruturados + agregação (CloudWatch/ELK).
- **Resiliência**: timeouts e retry com backoff no nó HTTP.
- **Privacidade**: retenção mínima de dados; criptografia em repouso (n8n encryption key).
- **Versionamento**: versionar flow/prompt e ativar _pair programming_ para mudanças.

---

## Roadmap (sugestão)

- Turnos de atendimento (SLA) + fila de prioridade por **urgência**.
- Enriquecimento de lead (verificação de e-mail/telefone).
- A/B de microcopy (chips e perguntas).
- Modo **estrito** do schema (bloquear propriedades extras).
- Integração de **analytics** de conversa (taxa de consentimento, drop-offs por fase).

---

## Arquivos Principais

- Flow: `flow/CHATBOT-ADEMICON-APOLLO.json`  
- Prompt: `prompt/consultor_ademicon_prompt_v1.2.json`

---

## Licença

Uso interno/privado da Ademicon e parceiros autorizados. Ajuste conforme política jurídica da organização.
