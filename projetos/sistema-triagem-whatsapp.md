# Projeto: Sistema de Triagem Inteligente — Barão Estamparia

> Status: especificação aprovada, implementação ainda não iniciada (decisão pendente: construir aqui via Claude Code ou em ferramenta externa/Codex).
> Fase 1 de uma visão maior (ver "Próximos passos" no fim).

## 1. Contexto e problema

A Barão Estamparia atende, hoje, majoritariamente times de futebol amador do ABC Paulista, por WhatsApp. O processo de "triagem" — entender o que o cliente quer (tipo de uniforme, conceito/ideia, referências visuais, quantidade, dados de contato) antes de passar pro designer criar a primeira arte — é 100% manual. É o maior gargalo identificado no negócio: consome o tempo de um atendente, e o ciclo de ida-e-volta até o designer começar a trabalhar com informação completa é longo.

Esse projeto é a primeira fase de uma visão maior (que inclui depois orçamento automático, pagamento, rastreamento de produção e portal do cliente), mas o escopo AQUI é só a triagem: do primeiro contato do cliente até uma "ficha" estruturada e completa, pronta pra um humano revisar e liberar pro design.

## 2. Objetivo

Um agente de IA conversa com o cliente no WhatsApp, conduz a entrevista de triagem (as mesmas perguntas que o atendente humano faz hoje), registra tudo de forma estruturada, e avisa o time interno quando a ficha estiver completa — sem precisar de um humano acompanhando a conversa em tempo real. Um humano só entra no fim, pra revisar a ficha e liberar (ou pedir mais informação, o que reabre a conversa automaticamente).

## 3. Visão do fluxo

**Fluxo do cliente:**
1. Cliente manda mensagem no número de WhatsApp dedicado a esse atendimento.
2. A IA se apresenta e conduz a conversa: tipo de produto/uniforme desejado → conceito/ideia (cores, estilo, referências) → pede fotos de referência se o cliente não mandou → quantidade aproximada → nome do time → dados de contato.
3. Se o cliente já manda várias informações de uma vez, a IA não repete pergunta já respondida.
4. Quando a ficha está completa, a IA confirma com o cliente e avisa que o time vai dar sequência.
5. Se faltar algo depois (humano pede mais info no painel), a IA volta a perguntar automaticamente, de forma natural.

**Fluxo interno:**
1. Time recebe um aviso (WhatsApp interno) quando uma ficha fica completa.
2. Abre o painel, vê a ficha estruturada + a conversa inteira + as imagens que o cliente mandou.
3. Decide: "Iniciar arte" (libera pro design) ou "Pedir mais info" (volta pra IA com uma nota do que falta).

**Regra de escopo da IA:** ela só trata de triagem. Perguntas sobre preço final, prazo de produção ou outros assuntos são respondidas com algo como "nosso time vai te passar isso já na sequência".

## 4. Arquitetura técnica

### 4.1 Conexão com WhatsApp: número quente, via conexão não-oficial

Em vez da API oficial da Meta (que exige verificação de empresa e pode levar dias), a decisão para este piloto é usar um **número já "quente"** (número real, com histórico de uso) conectado via integração não-oficial — simulando WhatsApp Web via QR code.

Duas formas práticas:
- **Evolution API** (open source, brasileira) hospedada em serviço gerenciado (Railway/Render). Sem custo de licença, mais controle; mas a sessão pode cair e exigir reconexão manual.
- **Z-API (ou SaaS similar)** — pago mensalmente, mas zero infraestrutura pra gerenciar, suporte do provedor.

**Risco a registrar:** integração não-oficial opera fora dos termos do WhatsApp — o número pode ser limitado/banido sem aviso. Mitigação: piloto de baixo volume, ritmo "humano" de envio, plano de migração pra API oficial da Meta se o canal se consolidar.

Recomendação dado "zero pessoa técnica no time": começar com **Z-API (ou equivalente gerenciado)**.

### 4.2 Stack

- **Next.js + Vercel** — app com rotas de API pros webhooks de mensagem + painel interno.
- **Supabase (Postgres + Storage + Auth)** — banco de dados, armazenamento de imagens, login do painel.
- **Anthropic API (Claude)** — motor de conversa, com tool use pra extração estruturada e visão nativa pra entender fotos de referência.

### 4.3 Modelo de dados (resumo)

- `clientes` — número de WhatsApp, nome, nome do time
- `conversas` — uma sessão de triagem por cliente
- `mensagens` — histórico completo, com id único pra nunca duplicar processamento
- `fichas_triagem` — tipo de produto, conceito/ideia, quantidade, nome do time, contato, observações, resumo da IA, status (`em_andamento` → `completa` → `em_revisao`/`precisa_mais_info` → `aprovada_para_arte`)
- `anexos` — imagens de referência, persistidas no Supabase Storage
- `perfis_internos` — pessoas do time com acesso ao painel

### 4.4 Fluxo do agente (passo a passo técnico)

1. Provedor de WhatsApp recebe mensagem e dispara webhook.
2. Aplicação identifica/cria cliente e conversa ativa.
3. Se vier imagem, baixa do provedor e salva no Supabase Storage.
4. Salva mensagem no histórico.
5. Carrega histórico + estado da ficha, chama a API da Claude com system prompt roteirizado + tools (`atualizar_ficha_triagem`, `marcar_ficha_completa`) + imagens.
6. IA responde e aciona tools; aplicação persiste atualizações.
7. Ficha completa → dispara aviso interno via WhatsApp.
8. Humano pede mais info → conversa reabre, IA pergunta automaticamente.

### 4.5 Painel interno mínimo

- Login (Supabase Auth) pras ~8 pessoas.
- Lista de fichas, filtrável por status.
- Detalhe: dados estruturados + conversa completa com imagens + botões "Iniciar arte" / "Pedir mais info".

## 5. Fases / cronograma sugerido

1. Setup de infraestrutura (provedor WhatsApp, Supabase, Vercel, Anthropic)
2. Banco de dados (migrations versionadas)
3. Integração de mensagens (teste de envio/recebimento)
4. Agente de IA (lógica de conversa + extração)
5. Painel interno
6. Piloto controlado (poucos clientes reais, ritmo humano)
7. Avaliação e decisão (manter número quente ou migrar pra API oficial)

## 6. Riscos e mitigações

| Risco | Mitigação |
|---|---|
| Número quente bloqueado/limitado | Piloto de baixo volume; número reserva; plano de migração pra API oficial |
| Queda de sessão (se auto-hospedado) | Preferir provedor gerenciado nesta fase |
| IA extrair informação errada | Toda ficha passa por revisão humana antes do design |
| IA responder fora do escopo | Regra explícita de redirecionar pro time humano |
| Manutenção sem time técnico | Código versionado em git, documentado, pensado pra sessões de dev assistido |

## 7. Custos estimados (ordem de grandeza — validar valores atuais)

- Provedor WhatsApp não-oficial: mensalidade fixa baixa
- Supabase: tier gratuito cobre o volume inicial
- Vercel: tier gratuito cobre o volume inicial
- Anthropic API: cobrança por uso, baixa pro volume de uma triagem por conversa

## 8. Setup necessário (checklist)

1. Escolher e contratar provedor de WhatsApp não-oficial
2. Conseguir o número quente e conectar via QR code
3. Criar projeto Supabase (banco + storage + auth)
4. Criar projeto Vercel conectado ao repositório
5. Criar conta e chave da API Anthropic
6. Cadastrar o time no Supabase Auth
7. Definir número/grupo interno pros avisos

## 9. Critérios de sucesso do piloto

- Tempo médio de triagem cai vs. processo manual
- Fichas chegam completas pro design na primeira vez
- Time opera o painel sem ajuda técnica
- Nenhum bloqueio do número durante o piloto

## 10. Próximos passos (fases futuras)

- Orçamento automático + cobrança + integração com Bling
- Gestão de produção com prazo calculado dinamicamente
- Portal de rastreamento pro cliente (estilo iFood/Correios)
- Avaliação de migração pra API oficial da Meta
