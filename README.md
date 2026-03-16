# Projeto EMII (Assistente Executiva Pessoal) - Workflows n8n

Este repositório contém a infraestrutura de automação, na forma de workflows do n8n, para a **EMII**, a assistente executiva pessoal de Márcio Aquino (CEO do Super ECO). Os workflows foram projetados com uma arquitetura escalável e são fortemente baseados nas melhores práticas de low-code/vibe code e dev ops para sistemas escaláveis de inteligência artificial.

## Arquitetura do Sistema

O sistema é construído utilizando uma arquitetura orientada a eventos e agentes, conhecida como Agentic Workflow. Nesse modelo, a orquestração central distribui as tarefas recebidas para sub-agentes especializados baseados na tecnologia LangChain. A infraestrutura integra-se nativamente com o Supabase para a persistência de dados, com a Z-API para comunicação via WhatsApp, e com as APIs do Google, especificamente Calendar e Tasks, para gestão da agenda.

Abaixo apresentamos o detalhamento de todos os 16 workflows que compõem o sistema, divididos por suas categorias funcionais.

## Fluxos de Entrada e Orquestração

Os fluxos de entrada são responsáveis por receber as mensagens externas, normalizá-las e decidir qual agente especializado deve processá-las.

| Workflow | ID | Função e Características Principais |
| :--- | :--- | :--- |
| **[DEV] EMII_WF01_GATE_INBOUND** | `CuyRRx0rht39rOFU` | Atua como o gate principal de entrada para mensagens via Webhook. Ele normaliza e valida payloads recebidos, extraindo informações de mídia e suportando a transcrição de áudio via AssemblyAI. Além disso, verifica a idempotência no Supabase para evitar processamento duplicado e constrói um envelope padronizado antes de encaminhar a requisição para o orquestrador. |
| **[DEV] EMII_WF02_MAIN_ORCHESTRATOR** | `yL10jX35B5h71Q9r` | É o cérebro do roteamento do sistema. Ele analisa a intenção do usuário utilizando o agente `Agent_Router` (alimentado por `gpt-5.2` com fallback para `gpt-4.1-mini`). Classifica as intenções entre conversacional, configuração, consulta, captura ou tarefas, roteando a execução para o sub-workflow apropriado. Possui um mecanismo de fallback de resposta para garantir que o usuário sempre receba um retorno. |
| **[DEV] EMII_01_InboundRouter (1)** | `DyTH1UnnRkaYPVh3` | Trata-se de um roteador alternativo ou versão legada robusta, composto por 72 nós. Ele gerencia o ciclo de vida completo de mensagens recebidas via Z-API, resolve condições de corrida e gerencia intenções pendentes ou que necessitam de clarificação. Constrói também contextos para LLMs interagindo diretamente com o Supabase. |

## Agentes Especializados (Sub-Workflows)

Uma vez que a intenção do usuário é identificada pelo orquestrador, a tarefa é delegada a um dos agentes especializados descritos abaixo.

| Workflow | ID | Função e Características Principais |
| :--- | :--- | :--- |
| **[DEV] EMII_WF03_AG_CONVERSATIONAL** | `3E5vYErpKq96I7iS` | Lida com interações de bate-papo geral, mantendo a personalidade característica da EMII. Utiliza o `Agent_Conversational` baseado em LangChain com o modelo `gpt-4.1-mini` e mantém a memória da conversa usando um nó de Buffer Window. |
| **[DEV] EMII_WF04_AG_QUERY** | `UODQp3FWJuXANRqy` | Responde a perguntas específicas sobre compromissos, tarefas e anotações. Para isso, busca itens armazenados no Supabase e no Google Calendar, utilizando o `Agent_Query` com modelos GPT avançados para interpretar a consulta e aplicar guardrails que garantem respostas precisas sobre a agenda. |
| **[DEV] EMII_WF04B_QUERY_FORMATTER** | `eDnggAlW92PmP53Y` | Responsável por formatar os dados brutos resultantes das consultas em mensagens amigáveis e naturais para o WhatsApp. Ele separa as notas da agenda, mascara dados sensíveis e utiliza o `Agent_QueryFormatter` para gerar o texto final baseado nas estruturas JSON. |
| **[DEV] EMII_WF05_AG_CONFIG** | `aP1qK5N0yM2vL8Zc` | Gerencia as preferências do usuário, bem como atualizações e exclusões de itens da agenda e tarefas. Executa exclusões lógicas no Supabase e sincroniza essas mudanças com o Google Calendar e Google Tasks. Utiliza pesquisa fuzzy com stopwords para localizar itens de forma inteligente no banco de dados. |
| **[DEV] EMII_WF06_AG_CAPTURE** | `Jk7Gkrd8tpclxOYx` | Extrai intenções de criação (como novos lembretes, tarefas ou notas) a partir da mensagem do usuário. É capaz de analisar a mensagem para criar múltiplos itens em lote, sincronizando novos compromissos com o Google Calendar e tarefas com o Google Tasks. Retorna perguntas de clarificação caso faltem dados essenciais. |
| **[DEV] EMII_WF08_AG_TASKS** | `nnXbUpGsk9jrh3OY` | Um agente focado exclusivamente em operações nativas do Google Tasks. Ele suporta ações diretas de listar tarefas pendentes e criar novas tarefas, interagindo diretamente com a API OAuth2 do Google Tasks. |

## Qualidade e Saída (Quality Control & Dispatch)

Antes de qualquer mensagem ser enviada de volta ao usuário, ela passa por um rigoroso controle de qualidade para garantir a segurança e o tom adequado.

| Workflow | ID | Função e Características Principais |
| :--- | :--- | :--- |
| **[DEV] EMII_WF10_QC_PIPELINE** | `cjf2oW9W4TOL71bh` | Atua como o Ethical Guardrail Agent (EGA). Ele revisa todas as respostas antes do envio, validando regras rígidas. Garante que o tom seja apropriado, claro e evita qualquer vazamento de prompts internos. Inclui um componente "Humanizer" que divide textos longos em múltiplas mensagens curtas, otimizando a leitura no WhatsApp. |
| **[DEV] EMII_WF20_DISPATCHER** | `CySQNloDhsLjAGhI` | É o despachante final de mensagens. Recebe a resposta já formatada e aprovada pelo pipeline de qualidade e realiza requisições HTTP para a Z-API para enviar o texto ao usuário. Além disso, registra o log de saída no Supabase para auditoria. |

## Agendadores e Observabilidade (Schedulers & Cron)

Para garantir que lembretes sejam enviados no momento correto e que o sistema esteja sempre operante, existem workflows que rodam de forma programada.

| Workflow | ID | Função e Características Principais |
| :--- | :--- | :--- |
| **[DEV] EMII_02_Scheduler** | `EhYQQqjn59Z3OUbx` | Agendador principal que verifica continuamente (a cada 1 minuto) os itens pendentes de notificação. Ele formata datas UTC para o fuso horário local do Brasil, dispara os lembretes via Z-API e atualiza o status de notificação no banco de dados. |
| **[DEV] EMII_02_ReminderScheduler** | `pfkbbIDZD9iB7K6Q` | Um fluxo complementar para garantir a robustez no envio de lembretes. Também roda a cada minuto, construindo mensagens ricas com emojis baseados no tipo de item (tarefa, evento, etc.) e implementando tratamento de falhas para evitar retentativas infinitas em caso de falha da Z-API. |
| **[DEV] EMII_05_Observability** | `mVMOxlPK86aLfD4l` | Responsável por monitorar a saúde do sistema EMII. Com um gatilho a cada 6 horas, busca mensagens recentes e itens pendentes no Supabase. Avalia essas condições e envia alertas de saúde para os administradores via Z-API caso o sistema apresente lentidão ou falhas. |

## Gestão de Conhecimento e Captura de Itens

Sistemas complementares para lidar com base de conhecimento e processamento em lote.

| Workflow | ID | Função e Características Principais |
| :--- | :--- | :--- |
| **[DEV] EMII_03_KnowledgeRetrieval_AGENTv1** | `483HlBwQ6p5qVzX9` | Agente de Retrieval-Augmented Generation (RAG) projetado para buscar informações em documentos ou bases de conhecimento. Implementa ferramentas LangChain como Wikipedia e Calculator, processa PDFs e documentos utilizando divisores de texto, e utiliza o Supabase Vector Store para busca semântica. |
| **[DEV] EMII_04_CaptureItems** | `qK9mRz2xYwP8vL5n` | Um fluxo legado ou complementar para processar itens em lote, focado em evitar duplicações. Ele itera sobre arrays de itens extraídos e verifica a duplicidade no banco de dados antes de efetuar a criação de novos registros. |

## Boas Práticas e Segurança Implementadas

Os workflows foram desenvolvidos seguindo padrões rigorosos de segurança e qualidade de código low-code:

**Credenciais Seguras:** Nenhuma chave de API ou token do Supabase ou Google está escrita diretamente em nós de código (`Code Nodes`). Todas as integrações utilizam os nós nativos do n8n com gerenciamento seguro de credenciais, garantindo que nenhum segredo seja exposto na exportação dos JSONs.

**Agentic Design:** O sistema faz uso extensivo dos nós `@n8n/n8n-nodes-langchain` com uma configuração padronizada em toda a arquitetura. A configuração padrão utiliza o modelo `gpt-5.2` como primário, com fallback para `gpt-4.1-mini`, sempre acompanhado de um Output Parser e um Retry Model para garantir respostas estruturadas e consistentes.

**Tratamento de Falhas e Guardrails:** Foi implementado um rigoroso controle de qualidade através do Ethical Guardrail Agent (EGA) no pipeline de QC. Esse mecanismo impede que respostas alucinadas, vazamentos de prompt ou mensagens com tom inapropriado sejam enviadas ao usuário final, assegurando uma experiência profissional e confiável.
