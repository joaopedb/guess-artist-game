# Regras do Jogo — Especificação Definitiva v1

> Decisões fechadas em 2026-11-18.
> Este é o documento **autoritativo** sobre as regras. Em caso de conflito com `product.md`, `stack.md` ou `architecture.md`, **este arquivo prevalece**.
>
> Visão do produto: `product.md` · Arquitetura: `architecture.md` · Stack: `stack.md`

---

## 1. Objetivo e escopo

Descobrir o **artista secreto do dia** na categoria escolhida, usando trechos progressivos de uma música e pistas comparativas a cada palpite.

- ✅ Sem login.
- ✅ Sem pontuação e sem ranking por enquanto.
- ✅ Com resumo compartilhável estilo *Termo*.
- ❌ Sem desafios de dias anteriores: **somente o dia atual** é jogável.

---

## 2. Ciclo diário e categorias

- **Virada:** 00:00 no fuso `America/Sao_Paulo`. Internamente, tudo é armazenado em UTC; a "data do jogo" é a data local.
- **Um desafio por `(data, categoria)`.**
- **Categorias:** Rock, Pop, Indie, Brasil e **Geral**.
  - "Geral" sorteia entre **todos os artistas elegíveis** do catálogo, independentemente do gênero.
- **Anti-repetição:** um mesmo artista **não** pode ser o secreto em duas categorias no mesmo dia. Pode voltar em outro dia.
- **Recomeço:** se o jogador limpar o cookie, pode começar de novo no mesmo dia. Sem ranking, isso não causa dano.

---

## 3. Estrutura da partida

- **6 tentativas** no máximo.
- Estado: `playing → won | lost`.
- Áudio liberado no início: **2 segundos**.

| Tentativa que errou | Próximo trecho liberado |
| :-- | :-- |
| 1ª | 4s |
| 2ª | 7s |
| 3ª | 11s |
| 4ª | 15s |
| 5ª | mantém 15s |
| 6ª | fim de jogo → revelação dos 30s |

- A **6ª tentativa** é a última: se errar, a partida termina e os 30s são revelados.
- **Não existe desfazer:** confirmar um artista consome a tentativa, mesmo que tenha sido engano.
- **Palpite repetido é bloqueado:** não gera nova linha, não consome tentativa e não libera nova dica.

---

## 4. Entrada do palpite

- Campo de **autocompletar** que consulta **apenas o banco local** — nenhuma chamada externa ao digitar.
- **Só artistas elegíveis** aparecem no autocomplete (ver §8). Nomes inexistentes ou incompletos não podem ser enviados.
- Busca em todas as categorias somadas, com normalização: minúsculas, sem acentos, sem espaços extras.
- **Nomes duplicados** são desambiguados com o país: ex.: `The Weeknd (EUA)`.
- Cada palpite válido registra: número da tentativa, artista escolhido, resultado colorido e timestamp.

---

## 5. Regras de comparação

Ordem fixa das colunas no grid:

| # | Campo | 🟢 Verde | 🟡 Amarelo | 🔴 Vermelho |
| :-- | :--- | :--- | :--- | :--- |
| 1 | Nome | Normalizado idêntico | — | Qualquer outro |
| 2 | Gênero / subgêneros | Conjunto idêntico | Interseção não vazia | Nenhuma tag em comum |
| 3 | Ano de estreia | Igual | Diferença de **1 a 5 anos** + seta | Diferença > 5 anos + seta |
| 4 | Ano da faixa | Igual | Diferença de **1 a 5 anos** + seta | Diferença > 5 anos + seta |
| 5 | País / origem | Mesmo país | Mesmo continente | Outro continente |
| 6 | Formação | Mesma categoria | — | Diferente |

### 5.1 Definições dos campos

- **Gêneros:** tags padronizadas em português. Artistas híbridos podem ter `["Rock", "Indie"]` e aparecer em ambas as categorias.
- **Ano de estreia:** ano do **primeiro álbum oficial** (MusicBrainz) — não o ano de formação da banda.
- **Ano da faixa:** ano de lançamento da música tocada, independentemente da carreira do artista.
- **País:** país de **formação ou nascimento** do artista, fixado na curadoria. Não muda com nacionalidade posterior.
- **Formação:** `solo | banda | dupla`.

### 5.2 Setas — regra definitiva

> **↑ = o ano do artista secreto é MAIOR que o do palpite** (secreto mais recente).
> **↓ = o ano do artista secreto é MENOR que o do palpite** (secreto mais antigo).

- Toda diferença de ano recebe uma seta, mesmo quando a cor for vermelha.
- Empate de ano: 🟢 verde e **sem seta**.
- Texto alternativo/ícone acompanha a seta para acessibilidade.

> ⚠️ Isso **inverte** a convenção descrita em `product.md` (que dizia ↑ = mais antigo). `game.md` prevalece.

### 5.3 Constantes configuráveis

Os limiares `1–5 anos`, o número de tentativas e a tabela de durações ficam em uma única estrutura `GAME_RULES` no back-end, coberta por testes.

---

## 6. Áudio progressivo

- Fonte: preview oficial de **30 segundos** da Apple (`previewUrl` do iTunes).
- O front reproduz do início e **para automaticamente** ao atingir o limite liberado.
- **Replays ilimitados** do trecho já liberado.
- Não é possível avançar além do limite pela interface. Um jogador com DevTools pode burlar — aceito, porque o segredo protegido é o **nome** do artista, não a música.
- **Falha nas faixas do iTunes:**
  - se **ninguém jogou ainda**, o desafio do dia é trocado por outro artista;
  - se a partida **já começou**, o jogador é avisado e segue sem áudio, usando apenas as pistas comparativas.
- Na revelação (vitória ou derrota), os **30s completos** são liberados.

---

## 7. Fim de jogo e compartilhamento

- **Vitória:** acertar o artista.
- **Derrota:** errar a 6ª tentativa.
- Revelação somente nesse momento: nome do artista, título da faixa, capa do álbum e áudio completo.
- Resumo compartilhável:
  - grid de cores das 6 colunas por tentativa;
  - contador `X/6`;
  - data e categoria.
- **O resumo NÃO revela o nome do artista secreto**, para não estragar o dia de quem receber.
- Não há pontuação nem ranking.

---

## 8. Elegibilidade do catálogo

Um artista só entra no sorteio e no autocomplete se tiver:

- nome normalizado (com país entre parênteses quando houver colisão);
- país e continente de formação/nascimento;
- formação (`solo | banda | dupla`);
- ano de estreia;
- ao menos uma tag de gênero padronizada;
- pelo menos uma faixa com `itunes_track_id`, `release_year` e marcação de principal.

Artistas com dados incompletos ficam **fora do sorteio e fora do autocomplete**.

---

## 9. Casos especiais

| Situação | Regra |
| :--- | :--- |
| Banda formada em um país com membros de outros | Usar país de **formação da banda** |
| Artista que mudou de nacionalidade | Usar país de **nascimento/formação**, fixo na curadoria |
| Dois artistas com o mesmo nome | Desambiguar com o país: `Nome (País)` |
| Faixa sem ano de lançamento | Excluída do sorteio |
| Artista com várias faixas válidas | Usar a `is_primary`; as demais servem como fallback |
| Empate em qualquer campo | Verde, sem seta |
| Mesmo artista em várias categorias | Pode aparecer em dias diferentes, nunca no mesmo dia |
| Todas as faixas falharem no iTunes | Trocar o desafio se ninguém jogou; se já começou, avisar e seguir sem áudio |
| Limpar cookie | Recomeço permitido no mesmo dia |

---

## 10. Status das decisões

**Fechadas nesta rodada:**
- [x] 6 tentativas, com revelação dos 30s na última
- [x] Palpite repetido bloqueado, sem consumo de tentativa
- [x] Sem desfazer: confirmação consome tentativa
- [x] Setas: ↑ = ano do secreto maior; ↓ = ano menor
- [x] Limiar de "próximo": 1–5 anos
- [x] Replays ilimitados
- [x] Resumo sem nome do secreto
- [x] Limpar cookie permite recomeçar
- [x] Categoria "Geral" = todos os elegíveis
- [x] Autocomplete só com artistas elegíveis
- [x] Sem desafios de dias anteriores
- [x] Fallback de áudio: trocar antes do 1º palpite; depois, seguir sem áudio
- [x] Nomes duplicados desambiguados com país
- [x] Sem pontuação e sem ranking
- [x] País de formação/nascimento, fixo

**Evoluções futuras possíveis (fora de escopo da v1):**
- ranking diário por apelido;
- histórico jogável;
- partidas multijogador;
- novas categorias;
- ajustes nos limiares de ano conforme dados reais.
