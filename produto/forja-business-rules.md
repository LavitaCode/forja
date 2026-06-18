---
id: forja-business-rules
type: specification
status: canonical
version: 1.2.0
created: 2026-06-17
updated: 2026-06-18
owners: [ayla, atena]
contributors: [atena, iris, artemis, hermes]
war-room: 2026-06-17-v2
related:
  - forja-story-map.md
  - forja-definicoes-finais.md
  - mockup-pricing-forja.html
---

# Forja — Regras de Negócio Canônicas

**Single source of truth para regras operacionais.**
Produto, Frontend, Backend e QA consultam este documento.
Story Map contém a jornada. Este documento contém as regras que governam cada etapa.

---

## Índice

→ [1. Definições](#1-definições)
→ [2. Tiers & Quotas](#2-tiers--quotas)
→ [3. Reset de Invocações](#3-reset-de-invocações)
→ [4. Memória & Contexto](#4-memória--contexto)
→ [5. Agentes por Tier](#5-agentes-por-tier)
→ [6. Roteamento de Modelo](#6-roteamento-de-modelo)
→ [7. Gates & Enforcement](#7-gates--enforcement)
→ [8. Estados de Erro & Recovery](#8-estados-de-erro--recovery)
→ [9. Edge Cases](#9-edge-cases)
→ [10. Compliance & LGPD](#10-compliance--lgpd)
→ [11. Decisões do War Room](#11-decisões-do-war-room)

---

## 1. Definições

| Termo | Definição |
|-------|-----------|
| **Invocação** | 1 chamada ao agente, iniciada com sucesso (POST /invoke aceito pelo backend). Streaming = 1 invocação. Cancelamento mid-flight pelo usuário = conta. Retry automático por erro 5xx do servidor = NÃO conta. Retry iniciado pelo usuário = conta. |
| **Mensagem** | 1 turno de conversa de um usuário comum (não-dev) com um agente especialista. Unidade de UX — internamente contabilizada em tokens. |
| **Token** | Unidade real de consumo. ~4 chars ≈ 1 token. É o que de fato é cobrado do backend. Tokens são invisíveis ao usuário — a UI mostra barra de uso. |
| **Sessão** | Conjunto de turns vinculados a um `session_id`. Persiste cross-sessão (dias, semanas). Uma sessão = um agente. |
| **Turn** | 1 par prompt+response dentro de uma sessão. Turn incrementa apenas após invoke bem-sucedido + memória salva. |
| **AHA Moment** | 2ª invocação onde o agente retoma contexto sem re-explicação do usuário (AHA-2). AHA-1 = sinalização na 1ª resposta ("I'll remember this"). |
| **Janela de uso** | Período no qual tokens são contados. Free = diário (00:00 às 23:59 UTC-3). Pro/Team = rolling por atividade. |
| **Barra de uso** | Indicador visual (UI) que mostra consumo de tokens relativo ao teto do tier. Sem número absoluto — só preenchimento proporcional. Estilo Claude. |
| **Tenant** | Organização/workspace de um usuário (org_id). Billing é por usuário, não por tenant. |
| **Agente default** | Ayla — primeiro agente que o usuário encontra. Orquestradora geral. |
| **Perfil dev** | Usuário técnico que invoca agentes para construir produto (CLI, War Room, code review). |
| **Perfil usuário comum** | Usuário não-técnico que conversa com agente especialista (Afrodite, Quiron, etc.) como serviço. |

---

## 2. Tiers & Quotas

### 2.1 Tabela de Tiers

| Tier | Teto diário/janela | Perfil dev | Perfil usuário comum | Preço | Estado |
|------|-------------------|-----------|----------------------|-------|--------|
| **Free** | 100k tokens/dia (dev) · 50k tokens/dia (usuário) | 20 invocações equiv. | ~30 msgs equiv. | R$0 | R0 (não enforced), R1 (enforced) |
| **Pro** | Janela 5h (token pool implícito) | Todos (15) agentes | Todos (15) agentes | R$99/mês/usuário | R0 (não enforced), R1 (enforced) |
| **Team** | Pool 8h compartilhado | Todos (15) + War Room | Todos (15) | R$119/assento/mês | R0 (não enforced), R1 (enforced) |

> **Nota R0 (hoje):** Nenhum limite é enforced. Tiers existem no produto mas sem billing counter implementado. Gate obrigatório para R1.
> **Rotação de modelos:** Qwen (~20x mais barato) absorve casual; Sonnet absorve técnico; Opus só Pro/Team heavy. Isso permite tetos generosos no Free sem comprometer margens.
> **Tokens invisíveis ao usuário:** UI exibe barra de uso (estilo Claude) — sem número absoluto. As equivalências "20 invocações" / "30 msgs" são orientativas e não aparecem na UI.

### 2.2 Free Tier — Detalhamento

**Perfil dev:**
- **Teto:** 100k tokens/dia (≈ 20 invocações técnicas médias de ~5k tokens cada)
- **Agentes disponíveis:** Ayla + acesso ao catálogo completo (outros com lock upgrade)
- **Reset:** Todo dia às 00:00 UTC-3 (horário de Brasília) — estilo Claude
- **Hard limit:** Ao atingir 100k tokens → HTTP 429 + barra vermelha + CTA upgrade
- **Degradação:** NÃO. Sem fallback para Qwen-only após limite. Hard block.
- **Modelo:** Qwen (casual) + Sonnet (técnico), rotação automática

**Perfil usuário comum:**
- **Teto:** 50k tokens/dia (≈ 30 mensagens casuais médias de ~1.5k tokens cada)
- **Agentes disponíveis:** 1 agente especialista (escolha na criação da conta — Afrodite, Quiron, etc.) + Ayla
- **Reset:** Todo dia às 00:00 UTC-3 — estilo Claude
- **Hard limit:** Ao atingir 50k tokens → 429 + barra vermelha + CTA upgrade
- **Modelo:** Qwen (casual padrão) + Sonnet se complexidade detectada

**Ambos:**
- **Barra de uso:** Visível na UI, preenche proporcionalmente ao consumo. Sem número de tokens.
- **Sinalização:** 70% preenchido → barra amarela. 90% → laranja. 100% → vermelha + bloqueio.
- **War room:** NÃO disponível no Free (Team tier)

### 2.3 Pro Tier — Detalhamento

- **Teto:** Janela rolling de 5h (token pool implícito — não exposto ao usuário)
- **Sonnet:** ~50 invocações técnicas por janela (ou equivalente em tokens)
- **Opus:** ~20 invocações heavy por janela
- **Reset:** 5h após última atividade (não meia-noite, não dia 1º)
- **Agentes:** Todos os 15 ativos — ambos perfis (dev e usuário comum)
- **Pausa no limite:** barra cheia + timestamp do reset + CTA Team
- **Modelo:** Qwen + Sonnet (default) + Opus (heavy tasks)
- **Barra de uso:** Mostra preenchimento da janela atual (reseta a cada 5h)

### 2.4 Team Tier — Detalhamento

- **Teto:** Pool de tokens compartilhado entre todos os assentos, janela de 8h
- **Pool Sonnet:** equiv. 150 invocações por janela
- **Pool Opus:** equiv. 60 invocações por janela
- **Reset:** 8h após última atividade de qualquer membro da org
- **Agentes:** Todos os 15 + War Room (orquestração multi-agente)
- **Pausa no limite:** barra cheia + timestamp do reset + notificação para o admin da org
- **Audit log:** Quem usou, qual agente, quando, tokens consumidos — visível para admin
- **Workspace compartilhado:** Sessões visíveis para todos os membros

---

## 3. Reset de Invocações

### 3.1 Free — Reset Diário

- **Quando:** Todo dia às 00:00 UTC-3 (meia-noite, horário de Brasília) — estilo Claude
- **Implementação:** Cron job diário zera o contador de tokens: `SET tokens_used = 0` onde `tier = FREE`.
- **Fuso padrão:** UTC-3 (Brasília). Se timezone do tenant não armazenado, usar UTC-3 como default.
- **Teto dev:** 100k tokens/dia. **Teto usuário comum:** 50k tokens/dia. Diferenciado por `user_profile` no tenant.
- **Notificação:** Nenhuma — reset é previsível (meia-noite todo dia). UI mostra horário do próximo reset ao atingir o limite.

### 3.2 Pro — Janela Rolling 5h

- **Quando:** 5h após a última invocação do usuário
- **Implementação:** Redis counter com TTL = 5h. Key: `quota:{userId}:sonnet` / `quota:{userId}:opus`. TTL reseta a cada invoke.
- **Exemplo:** Usuário invoca às 14:00 → janela expira às 19:00. Invoca às 16:00 → janela expira às 21:00.
- **Sem notificação automática:** CLI/Web mostra timestamp de reset após cada invoke.

### 3.3 Team — Pool Rolling 8h

- **Quando:** 8h após a última invocação de qualquer membro da org
- **Implementação:** Redis counter com TTL = 8h. Key: `quota:{orgId}:sonnet` / `quota:{orgId}:opus`.
- **Admin notificado:** Quando pool atingir 80% → alerta no dashboard admin.

### 3.4 Transições de Tier (Upgrade / Cancelamento voluntário)

- **Upgrade Free → Pro:** Reset imediato da janela (usuário ganha janela cheia). Goodwill.
- **Upgrade Pro → Team:** Reset imediato do pool. Goodwill.
- **Cancelamento voluntário Pro/Team:** Plano ativo até fim do período pago. Após expirar, conta entra em estado **BLOCKED** (não Free) — acesso ao histórico e sessões preservado, invocações bloqueadas. Usuário reativa inserindo novo cartão e volta ao tier anterior com goodwill (janela/pool cheia).
- **Sem downgrade automático para Free:** Downgrade destrói o moat (histórico de sessões, contexto acumulado) e é antipadrão punitivo. Block preserva tudo e mantém a conversão possível.

---

## 4. Memória & Contexto

### 4.1 Memory TTL por Tier

| Tier | TTL (inatividade) | Comportamento |
|------|-------------------|---------------|
| **Free** | 30 dias | Após 30 dias sem nenhuma invoke na sessão → soft archive (oculta da UI, mantém no banco) |
| **Pro** | 180 dias | Após 180 dias → soft archive |
| **Team** | 365 dias | Após 365 dias → soft archive |

- **Hard delete:** 180 dias após soft archive (independente do tier)
- **Recovery:** Possível até hard delete. Via dashboard (self-serve) ou suporte.
- **Transparência:** Usuário NÃO vê TTL explícito na UI (invisível). Se houver notificação de purge, avisar D-7 via email.

> **Rationale (Hermes):** TTL Free de 30 dias (não 7) — ICP tem ciclos erráticos, pode sumir por 2-3 semanas e voltar. TTL curto mataria o AHA e seria percebido como SaaS predatório. 30 dias cobre sprints normais de founder.

### 4.2 Contexto por Invoke (Session Window)

- **Regra base:** Últimos 20 turns da sessão
- **Teto de tokens por tier:**

| Tier | Max tokens de contexto |
|------|------------------------|
| **Free** | 32.000 tokens |
| **Pro** | 100.000 tokens |
| **Team** | 200.000 tokens |

- **Estimativa de tokens:** `aproximado = (prompt.length + response.length) / 4` (heurística simples)
- **Truncamento:** Se 20 turns > teto do tier → carregar os turns mais recentes que cabem no budget (prioridade decrescente por recência)
- **Aviso ao usuário:** Se sessão tem ≥15 turns → badge amarelo "Sessão longa". Se ≥25 turns → badge vermelho. Não bloqueia.
- **CLI:** Exibe aviso no output: `⚠️ Session has 18 turns. Consider starting a new session for better performance.`

> **Rationale (Hermes + Artemis):** Hermes: same context no free e pro = melhor DX. Artemis: teto técnico necessário (20 turns pode ser 100k tokens com code-heavy sessions). Compromisso: teto maior no free (32k) do que limitar contexto por tier.

### 4.3 Comportamento de Warm-up

- **Timeout de carga de memória:** 5s para carregar contexto do Neon.
- **Se timeout:** Fallback para sessão stateless (sem contexto anterior) + aviso ao usuário: "Contexto não carregado — continuando sem histórico da sessão."
- **Warm-up NÃO conta como invocação** se falhar (timeout ou erro de storage).

---

## 5. Agentes por Tier

### 5.1 Whitelist por Tier

| Tier | Agentes disponíveis |
|------|---------------------|
| **Free** | Ayla |
| **Pro** | Todos (15 ativos): Ayla, Apolo, Artemis, Atena, Afrodite, Gaia, Hefesto, Hera, Hermes, Íris, Musa, Quiron, Yara, Zeus, Atlas |
| **Team** | Todos (15) + War Room (orquestração multi-agente) |

### 5.2 Catálogo na UI

- **Free:** Mostra catálogo completo (15 agentes) com badge "Pro" nos bloqueados. Hover = tooltip de upgrade. Click = modal de upgrade.
- **Pro:** Mostra todos disponíveis. Badge "Team" no War Room.
- **Team:** Tudo desbloqueado.

> **Rationale (Hermes + Íris):** Catálogo visível com lock = transparência + FOMO específico (ICP técnico vê "Apolo para backend" bloqueado = motivação para upgrade). Esconder seria antipadrão (Vercel: usuários reclamam de "não sabia que tinha isso" post-churn).

### 5.3 Ayla como Default

- Ayla é o único agente no Free e o agente default ao criar nova sessão no Pro/Team.
- Razão: Ayla é orquestradora, conversa sobre múltiplos domínios, reduz friction de onboarding.
- Ayla pode sugerir especialista: "Para análise de backend aprofundada, o Apolo seria mais preciso. Quer trocar?" (Pro/Team).

### 5.4 Enforcement Técnico

```
app-layer (não RLS):
  if agentId NOT IN allowedAgents[plan]:
    → HTTP 403 "Agent not available on your plan"
    → Link upgrade
```

---

## 6. Roteamento de Modelo

### 6.1 Tabela de Roteamento

| Modelo | Tiers | Trigger |
|--------|-------|---------|
| **Qwen 2.5 Coder 32B** | Free, Pro, Team | Prompt <200 chars, sem código, sem keywords arquiteturais. Papo casual. |
| **Claude Sonnet 4.6** | Free (até quota), Pro (default), Team (default) | Prompt com código, arquitetura, decisões, refactor, review. >80% das invocações. |
| **Claude Opus 4.8** | Pro, Team (NÃO Free) | War room, refactor >500 linhas, handoff Ayla→Atena, task classificada como "heavy". |

### 6.2 Heurística de Roteamento

```
1. Detecta código (```, function, class, import, async)
2. Detecta arquitetura (architecture, design, refactor, ADR, migration, schema)
3. Detecta heavy task (war room, refactor all, 500+ lines, deploy)

Rota:
- Sem match → Qwen
- Match código/arquitetura + Free → Sonnet (até quota)
- Match código/arquitetura + Pro/Team → Sonnet
- Match heavy + Pro/Team → Opus
- Opus no Free → fallback Sonnet + aviso "Opus disponível no Pro"
```

### 6.3 Fallback de Modelo

- Opus indisponível → fallback Sonnet + `warning: "Using Sonnet (Opus unavailable)"`
- Sonnet indisponível → fallback Qwen + aviso `"degraded mode: using Qwen"`
- Qwen indisponível → fallback Sonnet (sem aviso de degradação)

---

## 7. Gates & Enforcement

### 7.1 Usage Gate (sequência obrigatória no invoke_agent)

```
POST /api/v1/invoke_agent
  │
  ├─ 1. Auth (JWT válido) → 401 se não autenticado
  ├─ 2. Plan check → effectivePlan(userId) ∈ {FREE, PRO, TEAM}
  ├─ 3. Agent whitelist → agentId ∈ allowedAgents[plan] → 403 se não
  ├─ 4. Usage check → count < maxPerWindow → 429 se excedido
  ├─ 5. Invoke agent (core/olympo)
  ├─ 6. Record usage (ATÔMICO) → fail-closed se falhar (não retorna resposta)
  └─ 7. Save turn na session_memory → fail-open se falhar (retorna resposta com aviso)
```

### 7.2 Atomicidade do Billing Counter

- **Implementação:** UPSERT PostgreSQL (INSERT ... ON CONFLICT UPDATE) sobre unique constraint `(tenantId, userId, scope)`
- **Race condition:** Edge-case na virada de janela (2 requests simultâneos) → usuário pode ganhar 1 request a mais. Impacto: benigno (fail-open no edge-case raro). Mitigação futura: `SELECT ... FOR UPDATE` se frequência aumentar.
- **Fail-closed:** Se `incrementUsage()` falhar → backend NÃO retorna resposta ao usuário. Log de `BILLING_INCONSISTENCY`. Usuário vê 500 com mensagem: "Billing error — request not counted. Try again."

### 7.3 Precedência de Gates (quando múltiplos limites aplicam)

Ordem de verificação (primeiro gate que falhar retorna):
1. Auth
2. Agent whitelist (tier)
3. Quota mensal (Free) ou quota da janela (Pro/Team)

### 7.4 Rate Limit por IP (Anti-Abuse Free)

- Máximo 3 signups por IP por hora
- Máximo 5 signups por IP por dia
- Implementação: Redis counter com TTL. Retorna 429 se excedido.
- Razão: Free com 10 invocações pode ser abusado (100 emails = 100 invocações/dia).

---

## 8. Estados de Erro & Recovery

### 8.1 Quota Exceeded

| Tier | Código | Mensagem | CTA |
|------|--------|----------|-----|
| Free dev (diário) | 429 | "Limite diário atingido. Renova à meia-noite." | "Upgrade to Pro (R$99/mês)" |
| Free usuário (diário) | 429 | "Limite diário atingido. Renova à meia-noite." | "Upgrade to Pro (R$99/mês)" |
| Pro (janela) | 429 | "Janela esgotada. Renova em {time}." | "Upgrade to Team" |
| Team (pool) | 429 | "Pool do time esgotado. Renova em {time}." | "Fale com o admin" |

**Sinalização preventiva via barra de uso (estilo Claude):**
- 70% da barra → barra amarela (sem mensagem intrusiva)
- 90% → barra laranja + tooltip discreto: "Quase no limite"
- 100% → barra vermelha + botão de ação desabilitado + CTA upgrade
- Sem contadores numéricos de tokens na UI — só a barra proporcional

### 8.2 Agent Not Available

| Código | Mensagem |
|--------|----------|
| 403 | "Agent '{name}' requires {tier} tier. [Upgrade →]" |

### 8.3 Memory Load Error

| Cenário | Comportamento |
|---------|---------------|
| Timeout (>5s) | Continua stateless + aviso: "Contexto não carregado — iniciando sessão sem histórico." |
| Storage error | Continua stateless + aviso: "Contexto indisponível — esta invocação não terá histórico da sessão." |
| Session not found | 404: "Sessão não encontrada. Inicie uma nova: `forja session new`" |
| TTL expirado (archived) | "Sessão arquivada (inativa há 30+ dias). [Restaurar →] ou iniciar nova." |

### 8.4 Billing Error

| Código | Mensagem | Comportamento |
|--------|----------|---------------|
| 500 | "Billing error — request not counted. Try again." | Fail-closed: não retorna resposta do agente |

### 8.5 Cartão Recusado (Stripe)

- **Grace period:** 7 dias após falha de cobrança
- **Retries automáticos:** D+0 (falha inicial), D+3, D+5 — 3 tentativas no total
- **Após 7 dias sem pagamento:** conta entra em estado **BLOCKED** — sem downgrade para Free. Invocações bloqueadas, histórico e sessões intactos.
- **Timeline de emails:**
  - D+0: "Falha no pagamento — atualize seu cartão para continuar usando o Forja."
  - D+3: "Seu acesso será bloqueado em 4 dias se o pagamento não for regularizado."
  - D+6: "Último aviso — bloqueio amanhã."
  - D+7: "Conta bloqueada. Regularize para reativar com tudo que você tinha."
- **Self-cure:** usuário atualiza cartão → desbloqueio imediato + tier restaurado + janela/pool cheia (goodwill)
- **Sem perda de dados:** sessões, memória e histórico são preservados durante o bloqueio.

---

## 9. Edge Cases

### 9.1 Definição de Invocação (gray areas)

| Cenário | Conta como invocação? |
|---------|----------------------|
| Streaming completo | Sim (1 invocação) |
| Usuário cancela mid-flight | Sim (request iniciado = conta) |
| Retry automático por 5xx do servidor | Não (erro do servidor) |
| Retry manual pelo usuário | Sim |
| Timeout de resposta (agente não respondeu) | Não (se não chegou resposta) |
| Warm-up de memória falhou (timeout) | Não conta se invoke não foi processado |

### 9.2 Upgrade Mid-Ciclo

- **Free → Pro:** Janela Pro começa imediatamente. Reset da cota Free. Goodwill.
- **Pro → Team:** Pool Team começa imediatamente. Sessões individuais tornam-se compartilhadas.
- **Não existe downgrade automático:** Ver §3.4 — cancelamento gera BLOCK, não Free.

### 9.3 Memória: Sessão Ativa vs Expirada

- Sessão ativa (usada nos últimos N dias per TTL do tier): carrega contexto normalmente
- Sessão arquivada (TTL expirado): UI mostra badge "Archived". Click = prompt de restore (grátis, re-ativa TTL)
- Restore automático: se usuário invoca com `--session <id>` de sessão arquivada → restore silencioso + continua
- Hard delete: 180d após archive. Irrecuperável. Notificação por email D-7

### 9.4 Limites Simultâneos

- Usuário Free atingiu quota mensal E conta está BLOCKED → quota mensal tem precedência (429 primeiro)
- Mensagem deve ser específica ao gate ativo (não genérica "upgrade")

### 9.5 Sessão Multi-Agente (Team)

- War room cria `session_id` compartilhado para todos os agentes da sala
- **1 mensagem do usuário ao War Room = 1 invoke do pool** — independente de quantos agentes especialistas a Ayla acionar internamente para responder. A orquestração interna é invisível para o billing.
- Timeout de agente no war room: 30s por agente. Se timeout → output parcial com marcação `[TIMEOUT: agent_name]`

---

## 10. Compliance & LGPD

### 10.1 Direito de Esquecimento

- **Solicitação:** Usuário pode deletar conta via dashboard (self-serve) ou email suporte@forja.dev
- **Prazo:** Hard delete em 48h após confirmação
- **Escopo:** session_memory, project_index_snapshot, subscription, user profile
- **Exceção:** Logs de billing (transações Stripe) retidos por 5 anos (obrigação fiscal BR)

### 10.2 Portabilidade de Dados

- Usuário pode exportar todas as sessões: `forja session export --all --format json`
- Export inclui: session_id, agent_id, turns (prompt + response), timestamps
- Export NÃO inclui: dados de billing, logs internos de infra

### 10.3 Localização de Dados

- Dados em: us-east-1 (Neon US) — atual
- Migração planejada: sa-east-1 (Neon SA) em Q3+ (após R1)
- Disclosure: presente na privacy policy e no signup

### 10.4 Auto-Archive & Purge

- D-7 antes de auto-archive: email "Sua sessão '{name}' será arquivada em 7 dias."
- D-7 antes de hard delete: email "Sua sessão arquivada '{name}' será deletada permanentemente em 7 dias. Exporte ou restaure."
- Restauração: sempre disponível até hard delete (self-serve)

---

## 11. Decisões do War Room

**War room:** 2026-06-17 v2 · Atena · Íris · Artemis · Hermes · Decisor: Ayla

| ID | Regra | Decisão | Rationale | Alternativa descartada |
|----|-------|---------|-----------|------------------------|
| BR-01 | Reset Free | **Diário (meia-noite)** — estilo Claude | Cria hábito diário + urgência de conversão. Dev: 100k tokens/dia. Usuário comum: 50k tokens/dia. Rotação Qwen/Sonnet/Opus mantém margem. | Mensal (baixa frequência de uso, sem urgência) |
| BR-02 | TTL Free | **30 dias** de inatividade | ICP tem ciclos erráticos. 7 dias mataria o AHA. 30d cobre sprints normais. | 7 dias (mataria moat), Infinito (COGS incontrolável) |
| BR-03 | Contexto | **Mesmo para todos** (gate por invocações, não por contexto) | Limitar contexto no free quebra o teste de valor real. ICP técnico detecta e desconfia. | Contexto menor no free (antipadrão Copilot) |
| BR-04 | Agente Free | **Ayla exclusivo**, catálogo visível com lock | Reduz friction, Ayla = rosto do produto. Catálogo visível = FOMO específico para upgrade. | Escolha de 1 entre 15 (friction), esconder catálogo (antipadrão) |
| BR-05 | 11ª invoke Free | **Hard block** (429), sem degradação | Free = acquisition tool, não produto degradado. Qwen-only dilui o moat e canibaliza Pro. | Qwen-only após limite |
| BR-06 | Janela Pro | **5h rolling** após última invoke | Sprints não são punidos (não perde janela no meio da sessão). Modelo Claude (familiar para ICP). | 5h fixed (midnight) |
| BR-07 | Catálogo UI | **Mostrar todos com lock** | Transparência + aspiração. "Não sabia que tinha isso" = churn post-launch. | Esconder locked (Vercel-style) |
| BR-08 | Billing fail | **Fail-closed**: não retorna resposta se billing não foi registrado | Billing inconsistente = dívida técnica acumulada. Fail-open criaria free usage infinito. | Fail-open + corrigir depois |
| BR-09 | Invocação def | **Request iniciado = conta**, exceto retry 5xx servidor | Previsível para o usuário. Alinhado com OpenAI (quem conta é a tentativa do usuário). | Só contar se response completa (complexo de implementar) |
| BR-10 | Cancelamento voluntário | **BLOCK após expirar período pago** (não downgrade) | Tier cancelado = conta em espera, não conta degradada. Histórico e sessões preservados. Reativação com goodwill. | Downgrade para Free (destrói moat, antipadrão punitivo) |
| BR-11 | Cartão recusado | **BLOCK após 7 dias** (D+0/D+3/D+5 retries; D+0/D+3/D+6/D+7 emails) | Sem perda de dados. Block é reversível com 1 clique. Downgrade exigiria re-upgrade e re-onboarding = fricção desnecessária. | Downgrade imediato para Free |

---

**Documento encerrado. Source of truth de regras de negócio do Forja.**
**Próxima revisão:** Com primeira implementação real de R1 (billing counter vivo).

Para implementação técnica: ver `forja-story-map.md` (seção 7 — Dependências) e `migrations/0068_forja_session_memory_billing_rls.sql` (proposta Artemis).
