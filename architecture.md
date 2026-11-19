# Arquitetura de Software — Jogo de Adivinhação Musical

> Status: rascunho para discussão · Data: 2026-11-18
> Stack: `stack.md` · Regras: `game.md` · Visão original: `product.md`
> As regras detalhadas prevalecem sobre a visão original em caso de conflito.

## 1. Contexto (C4 nível 1)

```
               ┌──────────────────┐
               │  Jogador anônimo │
               └────────┬─────────┘
                        │ HTTPS
                        v
┌───────────────────────────────────────────────────────────────┐
│                   Sistema do Jogo                             │
│  ┌───────────────────┐        ┌─────────────────────────────┐ │
│  │ Site (React/Vite) │  REST  │ API (FastAPI / Render Free) │ │
│  │ Cloudflare Pages  │<──────>│ autenticação anônima,       │ │
│  └───────────────────┘  JSON  │ comparação, desafio diário  │ │
│                               └──────────┬──────────────────┘ │
│                                          │ SQL                │
│                               ┌──────────v──────────┐         │
│                               │ Postgres (Supabase) │         │
│                               └─────────────────────┘         │
└───────────────────────────────────────────────────────────────┘
                        │ (somente no cache miss / pipeline)
                        v
          ┌──────────────────────────────────────────┐
          │ Fontes externas                          │
          │ iTunes Search · MusicBrainz · Spotify   │
          └──────────────────────────────────────────┘
```

**Atores:** jogador anônimo (browser), operador (roda o pipeline de curadoria), CI (testa e faz deploy).

---

## 2. Componentes e responsabilidades

| Componente | Responsabilidade | **Não** faz |
| :--- | :--- | :--- |
| **Site (React/Vite)** | Renderizar o grid de palpites, autocompletar, tocar o áudio progressivo, montar o texto de compartilhamento | Validar palpite, conhecer o artista secreto, calcular o resultado |
| **API (FastAPI)** | Criar desafios do dia, gerenciar sessão anônima, validar e comparar palpites, resolver/cachear URL de áudio | Servir HTML estático, guardar segredo no front, depender de login |
| **Postgres (Supabase)** | Fonte da verdade: catálogo, desafios, sessões, palpites, cache de URLs | Lógica de negócio complexa (evitar stored procedures por enquanto) |
| **Pipeline de curadoria** | Buscar, normalizar e validar metadados; produzir CSV revisável; importar no banco | Rodar em tempo real no request do jogador |
| **Cloudflare** | DNS, CDN, WAF, hospedagem do estático | Executar Python/FastAPI |

### Princípio arquitetural central
> **O segredo nunca cruza a fronteira da API.** O front recebe apenas projeções públicas: artistas do autocomplete, resultados coloridos, duração liberada de áudio e — no fim — a revelação. Toda decisão ("acertou?", "próxima dica") é tomada no servidor.

---

## 3. Modelo de dados

### 3.1 Tabelas

```text
genres
  id            serial PK
  slug          text UNIQUE          -- 'rock', 'brasil', 'geral'
  name_pt       text                 -- 'Rock', 'Brasil'
  active        boolean

artists
  id                serial PK
  name              text
  normalized_name   text UNIQUE INDEX -- minúsculas, sem acentos, para autocomplete
  country           text             -- 'Brasil', 'EUA', 'Reino Unido'
  continent         text             -- 'América do Sul', etc.
  formation         enum             -- 'solo' | 'banda' | 'dupla'
  debut_year        integer          -- 1º álbum oficial (MusicBrainz)
  created_at        timestamptz

artist_genres
  artist_id   FK -> artists
  genre_id    FK -> genres
  is_primary  boolean
  PK (artist_id, genre_id)
  -- um artista pode ter várias etiquetas: ['Rock', 'Indie']

tracks
  id               serial PK
  artist_id        FK -> artists
  title            text
  release_year     integer          -- ano da faixa tocada
  itunes_track_id  text UNIQUE      -- salvo no banco; URL de áudio é volátil
  cover_url        text             -- opcional; pode vir do Storage depois
  is_primary       boolean          -- faixa principal do artista

daily_challenges
  id               serial PK
  challenge_date   date             -- data local (America/Sao_Paulo)
  genre_id         FK -> genres
  track_id         FK -> tracks
  secret_artist_id FK -> artists
  created_at       timestamptz
  UNIQUE (challenge_date, genre_id)

daily_artist_usage                  -- garante: mesmo artista não em 2 gêneros no mesmo dia
  challenge_date   date
  artist_id        FK
  PK (challenge_date, artist_id)

game_sessions
  id               uuid PK           -- também é o valor opaco do cookie 'sid'
  challenge_date   date
  genre_id         FK
  status           enum             -- 'playing' | 'won' | 'lost'
  started_at       timestamptz
  updated_at       timestamptz
  UNIQUE (id)

guesses
  id                serial PK
  session_id        FK -> game_sessions
  attempt_number    integer
  guessed_artist_id FK -> artists
  result_json       jsonb            -- resultado colorido calculado no back
  created_at        timestamptz
  UNIQUE (session_id, attempt_number)

external_cache
  cache_key    text PK              -- ex.: 'itunes:track:123456'
  payload      jsonb                -- { "preview_url": "...", "cover": "..." }
  expires_at   timestamptz
```

### 3.2 Exemplo de `result_json` (contrato API → front)

```json
{
  "attempt": 2,
  "guessed": { "id": 42, "name": "Legião Urbana" },
  "fields": {
    "name":       { "status": "wrong",  "label": "Legião Urbana" },
    "genre":      { "status": "partial", "matched": ["Rock"] },
    "debut_year": { "status": "lower", "arrow": "down", "value": 1982 },
    "track_year": { "status": "higher", "arrow": "up", "value": 1985 },
    "country":    { "status": "same_continent", "label": "Brasil" },
    "formation":  { "status": "exact", "label": "Banda" }
  },
  "unlocked_seconds": 7,
  "solved": false,
  "finished": false
}
```

**Semântica das setas (decidida em `game.md`):** `↑` = o ano do artista secreto é **maior** (mais recente) que o do palpite; `↓` = o ano do artista secreto é **menor** (mais antigo). Isso inverte a convenção descrita em `product.md` — `game.md` prevalece. Cada seta vem com texto alternativo para acessibilidade.

---

## 4. Fluxos principais

### 4.1 Geração do desafio diário (preguiçosa / lazy)

1. Front pede `GET /api/game/today?genre=rock`.
2. API calcula a "data do jogo" em `America/Sao_Paulo` (não UTC).
3. Se não existe `daily_challenges` para `(data, gênero)`:
   - Transação com `SELECT ... FOR UPDATE` (ou advisory lock) para evitar corrida entre jogadores;
   - Escolhe um `track` cujo artista tenha o gênero e **não** esteja em `daily_artist_usage` na mesma data;
   - Insere o desafio e o registro de uso do artista.
4. API cria ou recupera a `game_sessions` pelo cookie `sid`.
5. Resposta pública: gênero, número de tentativas, `unlocked_seconds: 2`, lista de palpites vazia. **Nada sobre o segredo.**

> Alternativa futura: job agendado via GitHub Actions cron (não depende do Render, que não tem scheduler confiável no free tier) para gerar todos os gêneros à meia-noite. Lazy generation evita infra extra e é suficiente no início.

### 4.2 Palpite

```
[Front] POST /api/game/guess  { "genre": "rock", "artistId": 42 }
   │
   v
[API] 1. lê cookie 'sid' e valida assinatura
      2. carrega sessão + desafio do dia (data local)
      3. se sessão encerrada -> 409
      4. valida que artistId=42 existe no catálogo e é **elegível** (autocomplete é só UX)
      4b. se o artista já foi tentado nesta sessão -> 409, sem consumir tentativa nem liberar dica
      5. RECUPERA o secret_artist_id do banco
      6. compara os 6 campos (§5) => result_json
      7. grava em 'guesses' com attempt_number sequencial
      8. atualiza status da sessão se acertou / esgotou 6 tentativas
      9. devolve result_json (sem nome do secreto, a menos que tenha terminado)
   │
   v
[Front] atualiza grid, toca próximo trecho, exibe resumo se finished
```

### 4.3 Áudio progressivo

- A cada erro, o back devolve `unlocked_seconds`: `2 → 4 → 7 → 11 → 15`; ao terminar, `30` com a revelação.
- O front usa um único `<audio>` apontando para `/api/audio/{track_id}` e para a reprodução em `timeupdate` quando `currentTime >= unlocked_seconds`.
- O endpoint de áudio:
  1. Procura `external_cache['itunes:track:{id}']`;
  2. Se expirado, chama iTunes `lookup?id=...&entity=song` (com timeout e retry) e salva por 24h;
  3. Responde `302` ao `previewUrl` da Apple.
- **Fallback:** se a faixa principal falhar, o back tenta outra faixa marcada no catálogo para o mesmo artista.
- **Hardening opcional:** se o browser não respeitar o limite por `timeupdate`, trocar por endpoint que faz proxy com `Range` limitado. Isso não impede que alguém reconheça a música — o anti-trapaça do jogo protege o **nome do artista**, não a audição.

### 4.4 Fim de jogo e compartilhamento

- Condição de fim: acerto **ou** 6 tentativas.
- Nesse momento (e somente nele) a API passa a incluir `reveal: { artist, track, cover_url, preview_url }`.
- O front monta o resumo estilo Termo (grid de 🟢🟡🔴 + "3/6" + data + gênero) e copia para a área de transferência. Não precisa de endpoint de compartilhamento.

---

## 5. Regras de comparação (especificação das cores)

| Campo | 🟢 Verde | 🟡 Amarelo | 🔴 Vermelho |
| :--- | :--- | :--- | :--- |
| Nome | Normalizado idêntico | — | Qualquer outro |
| Gêneros | Conjunto de tags idêntico | Interseção não vazia | Nenhuma tag em comum |
| Ano de estreia | Igual | Diferença de 1–5 anos **+ seta** | Diferença > 5 anos **+ seta** |
| Ano da faixa | Igual | Diferença de 1–5 anos **+ seta** | Diferença > 5 anos **+ seta** |
| País | Mesmo país | Mesmo continente | Outro continente |
| Formação | Mesma categoria | — | Diferente |

> Os limites "1–5 anos" foram **decididos em `game.md`** e devem virar constantes configuráveis (`GAME_RULES`) no back, cobertas por `pytest`. Ajustes futuros dependem de dados reais do catálogo.

---

## 6. Segurança e anti-trapaça

- **Validação 100% server-side.** Ignorar qualquer campo enviado pelo front que represente resultado, duração ou acerto.
- **Cookie assinado** (`itsdangerous`) com `HttpOnly`, `SameSite=Lax`, `Secure` em produção; ID opaco (UUID) sem dados do jogador.
- **`Cache-Control: no-store`** nos endpoints de jogo para evitar respostas com revelação em cache de proxy/browser.
- **Rate limiting:** máximo de 1 palpite a cada 2s por sessão, 6 palpites por sessão/dia, 60 req/min por IP.
- **CORS** restrito ao domínio do Cloudflare Pages.
- **Secrets de infra** apenas em variáveis de ambiente do Render (Environment Variables do Web Service) e do pipeline; nada no bundle do front.
- **Logs:** nunca registrar o artista secreto; registrar apenas `session_id`, `genre`, `attempt` e latência.
- **Limitação conhecida e aceita:** DevTools permite inspecionar o DOM e o áudio. Isso é inevitável; o objetivo é impedir acesso direto à resposta no JS/API, não impedir que o jogador reconheça a música.

---

## 7. Requisitos não funcionais

| Item | Meta |
| :--- | :--- |
| Autocomplete | < 100 ms (consulta local com `pg_trgm`, sem chamada externa) |
| Palpite | < 200 ms (1 insert + 1 update no Postgres) |
| Fuso horário | `America/Sao_Paulo` para a virada do dia; armazenar tudo em UTC |
| Disponibilidade | Se o iTunes estiver fora: jogo segue com capa/texto; só o áudio degrada |
| Acessibilidade | Navegação por teclado, `aria-live` nos resultados, cor **+** ícone/texto (nunca só cor) |
| Privacidade | Sem login; cookie anônimo; retenção sugerida de 90 dias para sessões/palpites |
| Observabilidade | Endpoint `/health`, logs estruturados JSON, Sentry opcional depois |

---

## 8. Estrutura de pastas proposta (monorepo)

```text
SpotifyProject/
├─ apps/
│  ├─ web/                      # React + Vite + TypeScript
│  │  ├─ src/
│  │  │  ├─ components/         # GuessGrid, SearchBox, AudioPlayer, ShareCard
│  │  │  ├─ hooks/              # useGame, useAutocomplete
│  │  │  ├─ api/                # cliente HTTP tipado
│  │  │  └─ types/              # contratos compartilhados com a API
│  │  └─ vite.config.ts
│  └─ api/                      # FastAPI + Python 3.12
│     ├─ app/
│     │  ├─ main.py
│     │  ├─ routers/            # game.py, search.py, audio.py
│     │  ├─ domain/             # comparison.py, daily.py (regras puras)
│     │  ├─ models/             # SQLAlchemy
│     │  ├─ schemas/            # Pydantic
│     │  └─ infra/              # db.py, cache.py, security.py
│     ├─ alembic/
│     ├─ tests/
│     └─ Dockerfile
├─ scripts/catalog/              # fetch_*.py, normalize.py, import_catalog.py
├─ data/curated/                 # CSVs versionados
├─ .github/workflows/            # ci.yml, deploy.yml
├─ stack.md
├─ architecture.md
├─ game.md
└─ product.md
```

O código de comparação fica em `domain/comparison.py` como funções puras — é a parte mais testada do sistema.

---

## 9. Roadmap de implementação

| Fase | Entrega | Critério de pronto |
| :--- | :--- | :--- |
| F0 | Fundação: monorepo, Dockerfile, Alembic, CI | `pytest` e `vitest` passando no repo vazio |
| F1 | Catálogo mínimo: 3 gêneros, ~50 artistas via script + CSV | Import funciona; autocomplete responde |
| F2 | Loop do jogo: desafio diário + 6 palpites + grid colorido | Dá para jogar e ganhar localmente |
| F3 | Áudio progressivo + cache de 24h | 2s→4s→7s→11s→15s funcionando |
| F4 | Sessão anônima + anti-trapaça + rate limit | F5 no front não revela o secreto |
| F5 | Deploy: Pages + Render Free + Supabase + domínio | Jogo acessível por HTTPS (cold start aceito) |
| F6 | Compartilhamento, acessibilidade, polish | Resumo copiável; teclado e leitor de tela OK |
| F7 | Escalonar catálogo (200/gênero) e monitorar | Taxa de fallback de áudio conhecida |

---

## 10. Dívidas e decisões em aberto

**Decisões fechadas:** host da API (Render Free), Cloudflare (Pages/DNS/CDN), limiares de ano (1–5), sem ranking, capas via URL da Apple em `tracks.cover_url` (sem Storage), geração lazy do desafio, 6 tentativas, sem histórico jogável.

**Dívidas restantes (operacionais, não bloqueiam a implementação):**
1. Validar os limiares de 1–5 anos com dados reais do catálogo e ajustar `GAME_RULES` se necessário.
2. Definir a política de retenção de sessões/palpites (sugestão: 90 dias) — hoje os dados apenas se acumulam.
3. Avaliar job noturno de geração (GitHub Actions cron) se a geração lazy causar lentidão na virada do dia.
4. Avaliar ping agendado ou upgrade do Render se o cold start incomodar.
5. Medir a taxa de fallback de áudio após popular o catálogo; trocar faixas problemáticas no CSV.
