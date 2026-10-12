# FAST INVESTIMENTOS — Documentação operacional

Plataforma de grupos de investimento com apuração diária em dias úteis, painel do cliente e
backoffice administrativo. Stack: **FastAPI + MongoDB** e **Vite + React 19 + TypeScript**.

## 1. Estrutura

```
backend/
  server.py              bootstrap FastAPI, monta api_router (/api)
  lib/business.py        dias úteis BR, feriados, grupos, modalidades, cálculo de apuração
  lib/security.py        PBKDF2, sessões httpOnly, current_user/current_admin, auditoria, settings
  lib/db.py              handle motor + índices
  models/schemas.py      modelos Pydantic v2 (espelhados em frontend/src/lib/types.ts)
  routers/               auth, catalog, investments, withdrawals, admin
  seed.py                cria/atualiza o administrador e as configurações padrão
frontend/src/
  pages/                 Landing, Login, Cadastro, RecuperarSenha, Painel, Grupos, Checkout,
                         InvestimentoDetalhe, Saques, Historico, Admin, Legal
  components/            SiteHeader, AppShell, ProtectedRoute, InvestmentCard, WhatsAppButton, BrandLogo
  lib/                   api.ts (fetch tipado /api), types.ts, format.ts (BRL/máscaras), session.ts
```

## 2. Endpoints (`/api`)

| Método | Rota | Acesso | Descrição |
|---|---|---|---|
| GET | `/` | público | healthcheck |
| GET | `/catalog` | público | grupos, modalidades, WhatsApp/atendimento, data de referência |
| POST | `/auth/register` | público | cadastro (valida CPF, telefone, CEP, aceite) |
| POST | `/auth/login` | público | login, define cookie `fast_session` |
| POST | `/auth/logout` | sessão | encerra sessão |
| GET | `/auth/me` | sessão | usuário atual |
| POST | `/auth/forgot-password` | público | gera código de redefinição (30 min) |
| POST | `/auth/reset-password` | público | redefine senha e invalida sessões |
| POST | `/investments` | cliente | cria aporte (status `aguardando_pagamento`) |
| GET | `/investments` | cliente | lista do titular |
| GET | `/investments/dashboard` | cliente | agregados do painel |
| GET | `/investments/history` | cliente | aportes, rendimentos e resgates |
| GET | `/investments/{id}` | cliente/admin | detalhe + série por dia útil |
| GET | `/withdrawals/balance` | cliente | saldo disponível |
| POST | `/withdrawals` | cliente | solicita resgate |
| GET | `/withdrawals` | cliente | histórico de solicitações |
| GET | `/admin/summary` | admin | KPIs |
| GET | `/admin/clients?q=` | admin | clientes com busca por nome/e-mail/CPF |
| DELETE | `/admin/clients/{id}` | admin | exclusão LGPD do titular |
| GET | `/admin/investments?status=` | admin | todos os investimentos |
| POST | `/admin/investments/{id}/confirm-payment` | admin | confirma pagamento e ativa |
| POST | `/admin/investments/{id}/cancel` | admin | cancela com justificativa |
| GET | `/admin/withdrawals?status=` | admin | solicitações de resgate |
| POST | `/admin/withdrawals/{id}/approve` | admin | aprova com justificativa |
| POST | `/admin/withdrawals/{id}/reject` | admin | rejeita com justificativa |
| POST | `/admin/withdrawals/{id}/mark-paid` | admin | registra o pagamento efetuado |
| GET/PUT | `/admin/settings` | admin | link padrão, **links de checkout por grupo**, link de suporte, WhatsApp, e-mail, nome |
| GET | `/admin/audit` | admin | registros de auditoria |
| POST | `/cron/daily-accrual` | cron (Bearer) | apuração diária automática às 00h (America/Sao_Paulo) |

## 3. Regras de cálculo

- Grupos: GR 50, 100, 200, 250, 300, 400, 500, 700, 800, 1000 (valor fixo de entrada).
- Modalidades: 15 dias úteis → 15% no período; 30 dias úteis → 35% no período.
- Apuração **linear**: `rendimento_dia = amount × yield% ÷ dias_úteis`.
- Dias úteis: exclui sábados, domingos e feriados nacionais (fixos + móveis derivados da Páscoa).
- Referência temporal: `America/Sao_Paulo`, rollover às 00h — nenhum cálculo é feito no navegador.
- O período inicia no **primeiro dia útil após a confirmação do pagamento** pela administração.
- Estados de valor separados: **projetado**, **apurado**, **disponível**, **pago**.
- Investimento ativo que atinge a data de vencimento é marcado `concluido` pelo servidor, uma única vez.

## 4. Segurança e conformidade

- Senhas: PBKDF2-SHA256, 240.000 iterações, salt por usuário.
- Sessões: cookie `httpOnly`, `SameSite=Lax`, `Secure` (desligue com `COOKIE_SECURE=false` em HTTP local).
- Autorização por função validada **no servidor** (`current_admin`) — o frontend apenas reflete.
- Auditoria: cadastro, login, criação de aporte, confirmação de pagamento, decisões de resgate,
  alteração de parâmetros, exclusão de cliente.
- LGPD: política publicada em `/privacidade`, aceite obrigatório no cadastro, exclusão do titular no admin.
- A aprovação administrativa de um resgate **não** é confirmação de pagamento (etapa separada).

## 5. Configuração

`backend/.env`: `MONGO_URL`, `DB_NAME`, `CORS_ORIGINS`, `WEBHOOK_CRON_SECRET`, opcional `COOKIE_SECURE`.
O link de pagamento e o número/link de WhatsApp **não** ficam no `.env`: são editados em
`/admin` → aba **Configurações**, e aplicam-se imediatamente a toda a plataforma.

Na aba Configurações o administrador define:
- **Link de pagamento padrão** — usado por qualquer grupo sem link próprio.
- **Link de checkout por grupo** — um campo para cada um dos 10 grupos (GR 50 a GR 1000),
  permitindo um link de cobrança diferente por grupo. Em branco, o grupo usa o link padrão.
- **Link do suporte** — URL completa do atendimento (WhatsApp ou outro canal) usada pelo botão
  flutuante em todas as páginas. Em branco, o sistema monta o link a partir do número oficial.

### Apuração diária automática
`.emergent/crons.yml` agenda `POST /api/cron/daily-accrual` para **00h de Brasília, todos os dias**.
O endpoint exige `Authorization: Bearer $WEBHOOK_CRON_SECRET`, responde imediatamente e processa em
background: grava o apurado do dia de cada investimento, conclui os que venceram e registra a
execução na auditoria. É idempotente — entregas duplicadas e reexecuções no mesmo dia não geram
lançamentos repetidos.

Teste manual:
```bash
curl -X POST http://localhost:8001/api/cron/daily-accrual \
  -H "Authorization: Bearer $(grep WEBHOOK_CRON_SECRET backend/.env | cut -d= -f2)" \
  -H 'Content-Type: application/json' -d '{}'
```

## 6. Operação

```bash
cd /app/backend && python seed.py          # cria/atualiza o administrador
sudo supervisorctl restart backend frontend
sudo supervisorctl status                  # backend, frontend, mongodb
```

Logs: `/var/log/supervisor/backend.err.log`, `frontend.err.log`.
Backup do banco: `mongodump --uri "$MONGO_URL" --db app --out /backup/$(date +%F)`.

## 7. Testes

```bash
cd /app/backend && pytest                  # testes de API
cd /app/frontend && yarn typecheck         # tipagem estrita
```

Responsividade validada nas larguras 320, 360, 390, 768, 1024 e 1440 px: cartões empilham,
tabelas viram cartões no mobile, sem rolagem horizontal, botões com área de toque ≥ 44px e o
botão de WhatsApp não cobre conteúdo (páginas com `pb-28`).

## 8. Publicação

1. Ajuste `CORS_ORIGINS` para o domínio final e mantenha `COOKIE_SECURE=true` (HTTPS).
2. Rode `python seed.py` e troque a senha do administrador no primeiro acesso.
3. Em `/admin` → Configurações, informe o link oficial de pagamento e o WhatsApp de atendimento.
4. Publique pelo fluxo de deploy da plataforma; frontend e backend são servidos na mesma origem
   (`/api` no mesmo host), portanto nenhuma URL absoluta precisa ser configurada no frontend.
