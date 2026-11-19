# Stack Tecnológica — Jogo de Adivinhação Musical

> Status: **decisões fechadas** em 2026-11-18 · Este documento prevalece sobre `product.md` em temas de stack.
> Documentos complementares: `game.md` (regras) · `architecture.md` (arquitetura) · `product.md` (visão original)

## 1. Decisões consolidadas

| Tema | Decisão | Origem |
| :--- | :--- | :--- |
| Alvo de entrega | **Site web responsivo** (sem app desktop, sem PWA obrigatória) | Correção do usuário |
| Frontend | **React + Vite + TypeScript** | Usuário confirmou React/Vite e TypeScript |
| Backend | **Python 3.12 + FastAPI** | Usuário escolheu FastAPI |
| Banco de dados | **PostgreSQL gerenciado (Supabase)** | Usuário escolheu Postgres + já usa Supabase |
| Áudio | **iTunes Search API** com `trackId` salvo e URL resolvida sob demanda, cache de 24h | Usuário confirmou |
| Progresso do jogador | **Cookie anônimo opaco + estado no back-end** (sem login) | Usuário confirmou |
| Curadoria | **Script Python versionado + CSV revisável**, importado por CLI | Usuário confirmou |
| Front na nuvem | **Cloudflare Pages** (estático) | Usuário quer Cloudflare |
| API na nuvem | **Render Web Service (plano Free)** | Usuário confirmou familiaridade e aceitou o free tier por enquanto |
| DNS / CDN / WAF | **Cloudflare** na frente do domínio | Usuário quer Cloudflare |

**Idioma (decidido):** código, nomes de tabelas/funções e comentários técnicos em **inglês**; documentação, commits e textos de UI em **português**. O jogo é voltado ao público brasileiro.

---

## 2. Frontend

| Item | Escolha | Por quê |
| :--- | :--- | :--- |
| Linguagem | TypeScript 5.x | Evita bugs comparando anos, enums de cor e formato das respostas da API |
| Runtime | Node 22 LTS | Suporte longo; compatível com Vite e Cloudflare Pages |
| Framework | React 19 | Ecossistema maduro para autocomplete e estado de jogo |
| Build | Vite 6/7 | Build rápido, saída estática perfeita para Pages |
| Gerenciador de pacotes | npm | Definido para evitar ambiguidade de lockfile |
| Roteamento | React Router (leve) | URLs compartilháveis por gênero/dia (`/?genre=rock`) |
| Estado do servidor | TanStack Query v5 | Cache das respostas da API, refetch e estados de loading/error prontos |
| Estilo | Tailwind CSS v4 + CSS custom properties | Consistência de cores do grid (🟢🟡🔴) e responsividade rápida |
| Componentes | Próprios, sem biblioteca pesada | O jogo tem poucos componentes; evita bundle e dependência extra |
| Testes | Vitest + Testing Library | Testa a lógica de UI e o autocomplete sem browser |
| Deploy | Cloudflare Pages, `npm run build` → `dist/` | Estático, barato, CDN global |

**Restrição de ouro:** o front **nunca** recebe o nome/ID do artista secreto. Ele só recebe o resultado colorido de cada palpite e o estado público da partida.

---

## 3. Backend

| Item | Escolha | Por quê |
| :--- | :--- | :--- |
| Linguagem | Python 3.12 | Mesma linguagem do pipeline de curadoria; reuso de código |
| Framework | FastAPI 0.11x | Tipagem com Pydantic, OpenAPI automático, assíncrono |
| Servidor ASGI | Uvicorn (1 worker inicialmente) | Estado fica no banco, então escalar depois é trivial |
| Validação | Pydantic v2 | Contrato explícito de request/response |
| ORM / migrações | SQLAlchemy 2.0 (async) + Alembic | Modelo relacional claro; migrações versionadas |
| Driver | asyncpg | Melhor performance com Postgres |
| Cliente HTTP | httpx (async) | Timeouts e retry para iTunes/MusicBrainz no pipeline |
| Cache externo | Tabela `external_cache` no Postgres com `expires_at` (24h) | Evita um serviço Redis agora; migrar depois se latência virar problema |
| Sessão anônima | Cookie `sid` assinado (itsdangerous), `HttpOnly`, `SameSite=Lax`, `Secure` em prod | Sem login, mas resistente a manipulação pelo jogador |
| Rate limit | Middleware próprio ou `slowapi` | Limita palpites por sessão e requisições por IP |
| Lint / tipos | Ruff + mypy | Pega erros de typo em regras de comparação |
| Testes | pytest + pytest-asyncio + TestClient do FastAPI | Foco nas regras de comparação e validação server-side |

**Endpoint de áudio:** `/api/audio/{track_id}` resolve a URL do preview da Apple (cache 24h) e responde com `302` para o `previewUrl`. Se houver problema de CORS/range, o modo de fallback é proxyar o stream pelo próprio FastAPI (documentado em `architecture.md`).

---

## 4. Banco de dados (Supabase Postgres)

- **Postgres 16**, usando o pooler de transações do Supabase (`?pgbouncer=true`) para conexões do FastAPI.
- **pg_trgm** habilitado para autocomplete por similaridade de nome (`normalized_name`).
- **JSONB** para tags/subgêneros e para o resultado de cada palpite (`result_json`).
- **Alembic** para todas as mudanças de schema — nunca editar tabelas à mão em produção.
- Modelo resumido: `genres`, `artists`, `artist_genres`, `tracks`, `daily_challenges`, `game_sessions`, `guesses`, `external_cache`. Detalhes em `architecture.md` §3.

> **Decisão fechada:** usar **apenas o Postgres** do Supabase. A URL da capa fica em `tracks.cover_url` e é renovada junto com o cache de áudio. Auth, Storage e Edge Functions ficam de fora por enquanto.

---

## 5. Pipeline de curadoria (offline)

```
Spotify Web API (client credentials)  ─┐
Last.fm (fallback)                    ─┼─> scripts/catalog/fetch_*.py
MusicBrainz (1 req/s, User-Agent)     ─┤        │
iTunes Search API (trackId + capa)    ─┘        v
                                      data/curated/*.csv  (versionado no Git)
                                            │ revisão manual
                                            v
                              python -m scripts.catalog.import_catalog
                                            │
                                            v
                                  Postgres (Supabase)
```

- **Nenhuma chave de API em produção.** As chaves vivem apenas no ambiente do script (local ou CI de curadoria).
- **MusicBrainz:** respeitar rigorosamente 1 requisição/segundo + User-Agent com contato; cachear respostas em disco.
- **iTunes:** buscar faixas e capas; salvar `trackId`; **não baixar nem redistribuir áudio** — usar somente o `previewUrl` oficial da Apple.
- **Padronização:** subgêneros traduzidos para português e normalizados antes do CSV (ex.: `indie rock` → `Indie Rock`).
- **Revisão:** o CSV é o artefato auditável; nada entra no catálogo sem passar por ele.

---

## 6. Infraestrutura e deploy

```
                         ┌──────────────────────────┐
   Jogador ─────────────>│ Cloudflare (DNS/CDN/WAF) │
                         └────────────┬─────────────┘
                                      │
                    ┌─────────────────┴─────────────────┐
                    v                                   v
        ┌───────────────────────┐           ┌────────────────────────┐
        │ Cloudflare Pages      │           │ Render Web Service     │
        │ React/Vite estático   │─── API ──>│ FastAPI + Uvicorn      │
        │ jogo.seudominio.com   │  /api/*   │ api.seudominio.com     │
        └───────────────────────┘           └───────────┬────────────┘
                                                        │
                                                        v
                                          ┌────────────────────────┐
                                          │ Supabase Postgres      │
                                          │ (pooler de transações) │
                                          └────────────────────────┘
                                                        │ (cache miss)
                                                        v
                                          iTunes Search API (Apple)
```

### Por que Render (Free) e o papel do Cloudflare?
Você já usa Render e tem costume — isso pesa mais do que migração por conveniência. **Cloudflare Workers não roda FastAPI de verdade** (runtime JS/WASM; suporte a Python é experimental e incompatível com ASGI completo), então o Cloudflare fica com **Pages (front estático) + DNS/CDN/WAF**, e a API Python continua no Render. Isso mantém um único serviço de backend para operar.

### Limitações do Render Free (aceitas por enquanto)
| Limitação | Impacto no projeto | Mitigação |
| :--- | :--- | :--- |
| Serviço "dorme" após ~15 min de inatividade | **Cold start** de alguns segundos no primeiro acesso do dia | Aceitar no início; se incomodar, ping agendado via GitHub Actions ou subir para plano pago |
| ~750 horas/mês de runtime gratuito | Um serviço sempre ligado (~720 h/mês) cabe, mas sem folga para múltiplos serviços | Manter **um único** Web Service; pipeline roda local/CI, não no Render |
| Filesystem efêmero | Nada pode depender de disco local persistente | Já é o nosso caso: cache vai no Postgres, CSVs versionados no Git, capas na URL da Apple |
| Sem suporte a Docker Compose | Cada serviço é isolado | Não precisamos de compose em produção: front no Pages, API no Render, DB no Supabase |
| Recursos limitados (RAM/CPU) | Autocomplete e comparação são leves | Consultas com índice; evitar carregar catálogo inteiro em memória |

> Quando o projeto crescer (ou o cold start virar reclamação), a migração para um plano pago do Render não exige mudança de código — basta trocar o tipo de instância. Cloudflare e Supabase permanecem como estão.

### Variáveis de ambiente
- `DATABASE_URL` — string do Supabase com pooler (somente no back)
- `SESSION_SECRET` — chave para assinar o cookie (somente no back)
- `ALLOWED_ORIGINS` — domínio do Pages (CORS)
- `ITUNES_CACHE_TTL_SECONDS=86400`
- `ENV=local|staging|prod`
- Chaves de Spotify/Last.fm/MusicBrainz: **apenas no pipeline**, nunca no runtime

### CI (GitHub Actions — definido)
1. `pytest` + `pytest-asyncio` + `ruff` + `mypy` no `apps/api`
2. `vitest` + `tsc --noEmit` + `npm run build` no `apps/web`
3. Deploy automático: Pages (front) e Render (API) na branch principal

**Fora de escopo na v1:** testes E2E com Playwright (podem entrar depois, se o fluxo ficar instável).

---

## 7. Decisões rejeitadas (e por quê)

| Ideia | Motivo da rejeição |
| :--- | :--- |
| FastAPI dentro de Cloudflare Workers | Incompatível com o runtime do Workers; exigiria reescrever a API em TS |
| App desktop (Tauri/Electron) | Você confirmou que quer apenas site web |
| Spotify Web API em tempo real | Development Mode limita usuários; já descartado no doc de visão |
| Redis desde o dia 1 | Serviço extra sem necessidade; a tabela de cache atende |
| MongoDB | Comparações e agregações do jogo são relacionais |
| Hospedar/ baixar os áudios | Risco de direitos autorais + custo de banda; usar preview oficial |
| Autenticação com login | Fricção desnecessária para um jogo diário casual |
| Supabase Storage para capas | Decidido: só Postgres; a URL da Apple vai em `tracks.cover_url` |
| Playwright desde o dia 1 | Decidido: pytest + Vitest bastam para a v1 |
| CSS Modules / `fetch` puro | Decidido: Tailwind + TanStack Query |
| JavaScript sem tipos | Decidido: TypeScript no front |

---

## 8. Todas as decisões fechadas

- [x] Alvo: site web responsivo (sem desktop)
- [x] Front: React + Vite + TypeScript, npm, React Router, Tailwind, TanStack Query
- [x] Back: Python 3.12 + FastAPI, SQLAlchemy 2.0 async, Alembic, Pydantic v2
- [x] Banco: Supabase Postgres 16, sem Auth/Storage/Edge Functions
- [x] Áudio: iTunes + `trackId` + cache 24h no Postgres (`external_cache`)
- [x] Sessão: cookie anônimo assinado, estado no back-end
- [x] Curadoria: script Python versionado + CSV revisável
- [x] Deploy: Cloudflare Pages (front) + Render Free (API) + Cloudflare DNS/CDN
- [x] Idioma: código em inglês, docs/UI em português
- [x] Testes/CI: pytest + Vitest + ruff + mypy + tsc no GitHub Actions
- [x] Ranking: nenhum na v1 (ver `game.md`)

**Não há mais pontas soltas de stack.** Evoluções futuras possíveis (fora de escopo): Supabase Storage, Redis, Playwright, plano pago do Render e migração da API para outro host.

