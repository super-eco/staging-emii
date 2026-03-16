# Mega-Prompt de Engenharia: Construção da EMII (Substituição do n8n)

## Contexto do Projeto
Você é um Engenheiro de Software Full-Stack Sênior. Sua missão é reconstruir a **EMII (Assistente Executiva de IA)**, substituindo 16 workflows do n8n por uma arquitetura em código nativo.
O sistema deve manter **TODAS as integrações externas existentes** (Supabase, Z-API, OpenRouter, AssemblyAI, Google Calendar, Google Tasks). Apenas o "motor" (n8n) será substituído pelo código que você vai gerar.

## Arquitetura Desejada
- **Frontend (Lovable):** Dashboard em React + TailwindCSS para visualizar métricas, histórico de mensagens e tarefas.
- **Backend (Edge Functions / Node.js):** Serviços para receber webhooks da Z-API, processar orquestração com LangChain.js, interagir com Supabase e enviar respostas.
- **Banco de Dados:** Supabase PostgreSQL (Tabelas: `messages`, `items`, `pending_intents`, `user_prefs`). **O schema já existe, não altere as tabelas.**

---

## 1. Contratos de API e Webhooks (Camada de Entrada/Saída)

### Inbound Webhook (Z-API)
O sistema deve expor um endpoint POST `/webhook/zapi` que recebe mensagens do WhatsApp.
- **Idempotência:** O campo `body.messageId` deve ser usado para garantir que a mesma mensagem não seja processada duas vezes (verificar na tabela `messages`).
- **Resposta imediata:** O endpoint DEVE retornar `200 OK` imediatamente para a Z-API e enfileirar o processamento em background (para evitar timeouts).

### Outbound (Z-API)
O envio de mensagens deve ser feito via POST para:
`https://api.z-api.io/instances/{INSTANCE_ID}/token/{TOKEN}/send-text`
Body: `{"phone": "numero", "message": "texto"}`

---

## 2. Orquestração e Agentes (LangChain.js)

O core do sistema utiliza chamadas à API da OpenRouter (modelos `gpt-5.2` e `gpt-4.1-mini`). Abaixo estão os prompts exatos que devem ser implementados no código.

### 2.1 Agent Router (Orquestrador Principal)
**Objetivo:** Classificar a intenção do usuário.
**Modelo:** `gpt-5.2`
**Prompt de Sistema:**
```text
Você é EMII, a assistente executiva pessoal de Márcio Aquino, CEO da rede de supermercados Super ECO no oeste do Pará, atuando como Router Agent do EMII. Sua ÚNICA função é classificar a intenção do usuário em EXATAMENTE UMA das rotas válidas.

ROTAS VÁLIDAS:
- capture (PRIORIDADE ALTA): intenções de CRIAÇÃO ("me lembra de...", "anota que...", "preciso fazer...").
- query: CONSULTAS de informações ("o que tenho hoje?", "qual a senha do Wi-Fi?").
- conversational: interações SOCIAIS ("oi", "tudo bem?", "quem sou eu?").
- config: configurações e gerenciamento ("muda meu fuso", "cancela a reunião", "já fiz a tarefa").

REGRAS:
1. "lembrar", horário + ação futura, "anota" + conteúdo = capture
2. Perguntas pessoais/sociais = conversational
3. "contrasenha", conclusão ("já fiz"), cancelamento = config
4. Perguntas sobre notas salvas = query
```
**Schema de Saída (JSON Estrito):**
`{ "route": "conversational|query|capture|config", "confidence": number, "extracted_items": array }`

### 2.2 Agent Capture (Extração de Tarefas/Lembretes)
**Objetivo:** Extrair itens estruturados de intenções de criação.
**Prompt de Sistema:**
```text
Atue como Agente de Captura do EMII. Sua função é extrair itens da mensagem do usuário: lembretes, tarefas ou notas.
- reminder: lembrete ou compromisso com data/hora (sem usar a palavra "tarefa").
- task: quando o usuário usa a palavra "tarefa", "task", ou "to-do".
- note: anotação (quando contém "salve", "anota", "guarda").

REGRAS:
- Converta horários para UTC (GMT-3 + 3h).
- is_relative: true se for "daqui a X minutos", false se for horário absoluto.
- Se houver múltiplos pedidos (separados por "e"), extraia TODOS individualmente.
```
**Schema de Saída (JSON Estrito):**
Array de objetos com `type` (reminder/task/note), `title`, `event_datetime` (ISO UTC), `is_relative`, `duration_minutes`.

### 2.3 Agent Config (Gerenciamento de Itens)
**Objetivo:** Modificar, concluir ou cancelar itens.
**Prompt de Sistema:**
```text
Atue como Agente de Configuração do EMII. Classifique em um dos config_types:
- manage_item: gerenciar item existente. Requer item_action ("cancel", "done", "modify") e item_search (texto de busca).
- lead_time: alterar antecedência.
- master_password: definir contrasenha.
```
**Schema de Saída (JSON Estrito):**
`{ "config_type": string, "item_action": string, "item_search": string, "modify_fields": object }`

### 2.4 Agent Conversational (Bate-papo)
**Objetivo:** Respostas sociais curtas.
**Prompt de Sistema:**
```text
Você é EMII, assistente executiva.
REGRA DE OURO: TODA resposta deve ter NO MÁXIMO 3 LINHAS. Sem exceção.
Tom leve e humano, como conversa de WhatsApp. Usa emojis com moderação.
NUNCA invente informações. NUNCA mencione metadados técnicos.
```

### 2.5 Agent QC (Ethical Guardrail - Revisão Final)
**Objetivo:** Garantir a qualidade da resposta antes do envio.
**Prompt de Sistema:**
```text
Você é o EGA (Ethical Guardrail Agent). Revise a resposta.
CRITÉRIOS DE REJEIÇÃO (approved=false): Texto vazio, vazamento de prompt, conteúdo inapropriado, resposta incoerente com a pergunta (ex: lista de lembretes quando perguntou o nome).
Se approved=false, você DEVE fornecer final_text corrigido.
```
**Schema de Saída (JSON Estrito):**
`{ "approved": boolean, "final_text": string, "issues": array }`

---

## 3. Persistência (Supabase)

O código deve interagir com as seguintes tabelas via Supabase Client (`@supabase/supabase-js`):

- **messages**: Armazena histórico de conversas e deduplicação (campos: `id`, `provider_message_id`, `role`, `content`, `created_at`).
- **items**: Armazena tarefas, lembretes e notas (campos: `id`, `type` [reminder/task/note], `title`, `content`, `event_datetime`, `status` [pending/done/cancelled], `is_sensitive`).
- **user_prefs**: Preferências do usuário (campos: `user_phone`, `lead_time_minutes`, `master_password_hash`).

---

## 4. Integrações de Terceiros (A serem implementadas no Backend)

- **Google Calendar API:** Sincronizar itens do tipo `reminder`.
- **Google Tasks API:** Sincronizar itens do tipo `task`.
- **AssemblyAI:** Se o webhook da Z-API contiver `body.audio.audioUrl`, enviar a URL para transcrição via AssemblyAI antes de passar pelo Agent Router.

---

## 5. Instruções de Implementação para o Lovable

1. **Setup Inicial:** Crie o projeto React com TailwindCSS para o Dashboard.
2. **Dashboard UI:** Crie 4 abas:
   - *Histórico:* Tabela conectada a `messages`.
   - *Tarefas/Agenda:* Kanban ou Lista conectada a `items`.
   - *Configurações:* Gestão de `user_prefs`.
3. **Backend / Edge Functions:** Crie a estrutura de pastas para os serviços Node.js/TypeScript:
   - `/api/webhook`: Recebimento da Z-API.
   - `/services/orchestrator.ts`: Implementação do Agent Router usando `LangChain.js`.
   - `/services/agents/`: Arquivos separados para Capture, Config, Query, Conversational e QC.
   - `/services/supabase.ts`: Client de banco de dados.
4. **Fluxo de Execução:**
   Webhook -> Deduplicação (Supabase) -> Transcrição (se áudio) -> Agent Router -> Agente Específico (Capture/Query/Config/Conv) -> Agent QC -> Z-API Outbound.

**Aja como um engenheiro autônomo. Gere a estrutura completa de código, garantindo que as regras de prompt e schemas de saída acima sejam respeitadas rigorosamente nas chamadas LLM.**
