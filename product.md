# Documento de Visão e Decisões Técnicas — Jogo de Adivinhação Musical

> ⚠️ Este é o documento de **visão original**. As regras detalhadas e atualizadas ficam em `game.md`, que prevalece em caso de conflito — em especial na semântica das setas de ano e na origem dos artistas do autocomplete.

## 1. Visão Geral do Produto
Um jogo web diário de adivinhação musical em português, inspirado na mecânica de investigação por dicas visuais do **Gamedle / Spotle** combinada com a reprodução progressiva de áudio do **Heardle**.

* **Objetivo Principal:** O jogador escolhe uma categoria musical (gênero) e tenta descobrir qual é o **artista secreto do dia** ouvindo trechos curtos de uma música e analisando pistas comparativas a cada palpite errado.
* **Diferenciais:**
  * **Sem barreira de login:** Não exige autenticação via Spotify nem limita o acesso a grupos fechados de teste.
  * **Catálogo Híbrido (Brasil + Internacional):** Foco no público brasileiro, contemplando estilos nacionais (MPB, Rock Nacional, Sertanejo, Rap BR, etc.) e internacionais.
  * **Múltiplas trilhas diárias:** Em vez de apenas um desafio global por dia, o usuário pode escolher jogar desafios separados por gênero musical (ex: Rock, Pop, Indie, Brasil, Geral).

---

## 2. Mecânica e Regras do Jogo

### 2.1. Fluxo da Partida
1. O jogador acessa o site e seleciona a **categoria/gênero** que deseja jogar no dia.
2. O player inicia liberando apenas um trecho curto da música (ex: **2 segundos**).
3. O jogador digita o nome de um artista no campo de busca (autocompletar) e envia seu palpite.
4. **Se errar:**
   * O sistema exibe uma linha comparando o **Artista Chutado** com o **Artista Secreto** usando códigos de cores (Verde = exato, Amarelo = próximo/parcial, Vermelho = incorreto) e setas direcionais ($\uparrow$ / $\downarrow$).
   * Um trecho maior da música é desbloqueado (ex: $2\text{s} \rightarrow 4\text{s} \rightarrow 7\text{s} \rightarrow 11\text{s} \rightarrow 15\text{s}$).
5. **Se acertar (ou esgotar as tentativas):** O artista secreto, a capa do álbum e o trecho completo de 30 segundos são revelados, gerando um resumo compartilhável (estilo *Termo*).

### 2.2. Colunas de Pistas (Enigmas)
A cada tentativa, os seguintes atributos são comparados entre o palpite e a resposta correta:
* **Gênero / Subgêneros:** Indica se há correspondência exata ou parcial entre as etiquetas de estilo do artista.
* **Ano de Estreia do Artista:** Ano de lançamento do primeiro álbum oficial (com indicador se o artista secreto é mais antigo $\uparrow$ ou mais recente $\downarrow$).
* **Ano da Faixa:** Ano de lançamento da música específica que está tocando no desafio (com indicador $\uparrow$ ou $\downarrow$).
* **País / Origem:** Nacionalidade ou país de formação do artista (amarelo caso seja do mesmo continente).
* **Formação:** Classificação entre Banda, Artista Solo ou Dupla.

---

## 3. Arquitetura e Fontes de Dados

Para evitar bloqueios comerciais, limites de usuários (*Development Mode* do Spotify) e instabilidade em tempo real, adotou-se uma arquitetura híbrida:

* **Curadoria Offline (Script de População):**
  * **Lista Inicial:** Coleta dos ~200 artistas mais populares por gênero musical (via Spotify Web API *Client Credentials* ou Last.fm).
  * **Metadados (*MusicBrainz API*):** Consulta automatizada (respeitando o limite de 1 requisição/segundo) para extrair país de origem, ano de estreia, tipo de formação e *tags* de subgêneros.
  * **Áudio e Capas (*iTunes Search API*):** Busca das faixas principais de cada artista para registrar os identificadores (`trackId`) e capas dos álbuns.
* **Banco de Dados Próprio:**
  * Armazena os artistas já higienizados, com subgêneros padronizados em português e suporte a **múltiplos gêneros por artista** (ex: um artista marcado como `["Rock", "Indie"]` pode ser sorteado em ambas as categorias, desde que não no mesmo dia).

---

## 4. Decisões Técnicas para "Pontas Soltas"

| Desafio Identificado | Decisão Tomada | Justificativa |
| :--- | :--- | :--- |
| **Expiração dos links de áudio do iTunes** | Salvar apenas o `trackId` no banco e consultar a URL do áudio sob demanda com **cache de 24h no servidor**. | Evita links quebrados/mudos no futuro sem sobrecarregar a API da Apple a cada acesso de jogador. |
| **Ambiguidade de datas na carreira** | Exibir **dois indicadores de ano**: *Ano de Estreia (1º álbum)* e *Ano da Música tocada*. | Elimina confusões (ex: artistas que começaram em bandas nos anos 60, mas têm hits solo nos anos 80). |
| **Artistas híbridos** | Permitir múltiplas etiquetas de gênero por artista no catálogo. | Garante que artistas de fronteira (ex: *Indie Rock* ou *Pop Rock*) apareçam naturalmente nas respectivas categorias. |
| **Chutes de artistas inexistentes** | O campo de autocompletar busca **exclusivamente na base local completa** (todos os gêneros somados). | Garante resposta instantânea sem chamadas externas ao digitar e assegura que todo palpite tenha metadados para comparação. |
| **Segurança contra trapaças (`F12`)** | Validação dos palpites feita **100% no back-end**. | O front-end recebe apenas o áudio e o resultado colorido de cada tentativa, ocultando o ID e o nome do artista secreto até o fim do jogo. |