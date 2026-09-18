# Academy Support Bot — Como funciona

## O que é

Bot de Discord da Asimov Academy que automatiza o suporte a alunos. Ele busca perguntas deixadas nos comentários das aulas na plataforma WordPress, distribui cada comentário para um professor responsável, posta as pendências no Discord em intervalos regulares e gera uma resposta rascunho com IA para cada dúvida.

---

## Visão geral da arquitetura

```
Discord Gateway
      │
  SupportBot (bot.py)
      │
  setup_hook() ──── carrega 4 cogs sequencialmente
      │
  ┌──────────────────────────────────────┐
  │ TeachersCog                          │
  │   cog_load() → Teachers.load_spreadsheet()  │
  │   (busca professores no Google Sheets)      │
  └──────────────────────────────────────┘
      │
  ┌──────────────────────────────────────────────────────────┐
  │ CommentsCog  (task loop a cada 12 minutos)               │
  │                                                          │
  │  BotComments.post_available_comments_to_channel()        │
  │    │                                                     │
  │    ├─ CommentHandler.reload_comments()                   │
  │    │     └─ Thread → load_comments()                     │
  │    │           ├─ py_wordpress_api (busca comentários)   │
  │    │           ├─ preprocess + merge com DB local        │
  │    │           ├─ assign_teachers()                      │
  │    │           └─ assign_comment_status()                │
  │    │                                                     │
  │    ├─ get_comments_to_post()                             │
  │    ├─ BotAI.answer(comentário) → LangChain + GPT-4o-mini │
  │    ├─ save_bot_comment()                                 │
  │    └─ post_comments() → mensagens no Discord             │
  └──────────────────────────────────────────────────────────┘
      │
  Slash commands (sob demanda):
    /my_comments       → pendências do professor que chamou
    /pending_comments  → resumo de todas as pendências
    /dump_pending      → lista completa de pendências
    /close_comment     → fecha um comentário manualmente
    /teachers          → lista de professores ativos
    /load_spreadsheet  → recarrega dados do Sheets
    /question          → menciona um professor aleatório
    /help              → lista todos os comandos
    /uptime            → tempo de execução do bot
```

---

## Estrutura de arquivos

```
academy_support_bot/
├── bot.py                          # Ponto de entrada, SupportBot
├── comment_handler.py              # Gerenciador do banco de dados de comentários
├── comment_processing.py           # ETL: busca, normaliza, mescla, classifica comentários
├── async_functions.py              # Formata e envia mensagens no Discord
│
├── cogs/
│   ├── comments_cog.py             # Task periódica + comandos de comentários
│   ├── teachers_cog.py             # Comandos de gerenciamento de professores
│   ├── help_cog.py                 # /help
│   └── uptime_cog.py               # /uptime
│
├── features/
│   ├── bot_ai/comment_answer.py    # Classificador + gerador de respostas (LangChain)
│   ├── bot_comments/bot_comments.py # Orquestrador do fluxo de comentários
│   ├── bot_env/bot_env.py          # Singleton: variáveis de ambiente
│   ├── bot_logger/bot_logger.py    # Singleton: log rotativo em arquivo
│   ├── bot_pidfile/bot_pidfile.py  # Guarda de instância única via PID file
│   ├── chatgpt/chatgpt.py          # Cliente OpenAI direto (saída estruturada)
│   ├── help/help.py                # Singleton: registro de comandos para /help
│   ├── simple_doc/simple_doc.py    # Gerador de .docx a partir de respostas da IA
│   ├── teacher/teacher.py          # Modelo Teacher + Singleton Teachers
│   └── uptime/uptime.py            # Calcula tempo de execução do processo
│
├── utils/
│   ├── singleton.py                # Metaclasse Singleton usada em features/
│   └── worker.py                   # Ponte entre código bloqueante e asyncio
│
└── dash/
    ├── app.py                      # Painel Streamlit para controlar o bot
    └── run_bot_forever.py          # Lançador de processo + loop de reinício
```

---

## Módulos principais

### `bot.py` — Ponto de entrada

Define `SupportBot(commands.Bot)`. No `setup_hook()`, os quatro cogs são carregados em ordem com 1 segundo de intervalo entre eles; em seguida, os slash commands são sincronizados com o servidor Discord configurado (não globalmente, para agilizar atualizações). O bot só inicia se `is_single_instance()` passar — um PID file em `~/.academy_support_bot/` impede dois processos rodando ao mesmo tempo.

**Variáveis de ambiente obrigatórias** (carregadas via `.env`):

| Variável | Uso |
|---|---|
| `DISCORD_TOKEN` | Autenticação no Discord |
| `DISCORD_SERVER_ID` | Guild onde os slash commands são registrados |
| `DISCORD_CHANNEL_ID` | Canal onde os comentários são postados |
| `OPENAI_API_KEY` | API da OpenAI (LangChain + ChatGpt) |
| `WP_USERNAME` / `WP_PASSWORD` | Acesso à API REST do WordPress |
| `BOT_DEV` (opcional) | Usa planilha de teste do Google Sheets |

---

### `comment_processing.py` — Pipeline ETL

É o coração do bot. Toda vez que `CommentHandler.reload_comments()` é chamado, este módulo executa o seguinte fluxo:

```
1. fetch_wordpress_comments()      ← py_wordpress_api
        │
2. preprocess_wordpress_comments() ← normaliza colunas, parseia HTML, resolve post_id → nome do curso
        │
3. load_local_comments()           ← lê o parquet local
        │
4. combine_comments()              ← mescla novos comentários sem sobrescrever os existentes
        │
5. assign_teachers()               ← distribui comentários para professores
        │
6. assign_comment_status()         ← determina quais comentários estão pendentes
        │
7. Salva no parquet                ← persiste o estado atualizado
```

**`assign_teachers()`**

Para cada comentário sem professor atribuído, busca os professores que cobrem aquele curso (`Teachers.get_teachers_by_content()`). A correspondência usa distância de Levenshtein (tolerância de até 4 caracteres de diferença). Se um professor já respondeu em uma thread, todos os outros comentários da mesma thread recebem o mesmo professor.

**`assign_comment_status()`**

Um comentário é marcado como `answer_pending = True` quando:
- Foi feito por um aluno (não é WordPress ID de professor)
- Não tem nenhuma resposta de professor ainda
- Não foi fechado manualmente

Se o último comentário de uma thread é de um professor, toda a thread é marcada como fechada.

---

### `comment_handler.py` — Banco de dados de comentários

Gerencia o DataFrame pandas em memória e o arquivo parquet em disco (`~/.academy_support_bot/bot.db`). É a fonte de verdade do estado dos comentários durante a sessão.

**Métodos principais:**

| Método | O que faz |
|---|---|
| `reload_comments()` | Roda `load_comments()` em thread separada; atualiza `comments_df` |
| `get_comments_to_post()` | Retorna pendentes ainda não postados nessa sessão |
| `mark_as_posted(ids)` | Adiciona IDs ao conjunto de já postados |
| `close_comment(id)` | Fecha o comentário e toda a thread; salva no parquet |
| `save_bot_comment(id, texto)` | Salva a resposta gerada pela IA; salva no parquet |

O conjunto `posted_comment_ids` existe apenas em memória — ao reiniciar o bot, todos os pendentes são elegíveis para repostagem.

---

### `async_functions.py` — Formatação e envio no Discord

`post_comments(comments_df, channel)` formata cada comentário como mensagem Markdown e envia ao canal. A mensagem inclui:
- Menção ao professor (`<@discord_id>`)
- ID do comentário, timestamp, nome do curso, nome do aluno
- Prévia da mensagem (máximo 1400 caracteres)
- Resposta rascunho da IA, se disponível (enviada como segunda mensagem)

Entre cada mensagem há um `await asyncio.sleep(2)` para respeitar o rate limit do Discord.

`post_comment_summary()` envia uma mensagem de resumo com total de pendências, quebrado por professor e por curso.

---

### `features/teacher/teacher.py` — Modelo de professores

**`Teacher` (dataclass)**

| Campo | Tipo | Descrição |
|---|---|---|
| `name` | `str` | Nome do professor |
| `wp_id` | `int` | ID do professor no WordPress |
| `discord_id` | `int` | ID do professor no Discord |
| `contents` | `list[str]` | Cursos/projetos atribuídos |
| `question` | `bool` | Disponível para questões avulsas via `/question` |

`is_content_assigned(content, exact=False)` usa distância de Levenshtein para comparar `content` com cada item de `contents`. Com `exact=False`, tolera até 4 caracteres de diferença.

**`Teachers` (Singleton)**

Carregado uma vez na inicialização do bot via `load_spreadsheet()`. Lê quatro abas do Google Sheets:
- `teachers` — lista de professores ativos
- `previous_teachers` — professores inativos (usados para identificar respostas de professores em comentários antigos)
- `existing_courses` — matriz de cursos × professores
- `existing_projects` — matriz de projetos × professores

A autenticação usa o arquivo `~/academy-support-bot-66332ff89041.json` (service account do Google).

---

### `features/bot_comments/bot_comments.py` — Orquestrador

Camada intermediária entre os cogs e os módulos de baixo nível. O método central é `post_available_comments_to_channel(channel)`, que executa na ordem:

1. `comment_handler.reload_comments()` — recarrega comentários do WordPress
2. `get_comments_to_post()` — filtra os que ainda não foram postados nessa sessão
3. `_add_ai_comments()` — gera respostas com IA para os sem resposta
4. `post_comments()` — envia ao canal Discord

---

### `features/bot_ai/comment_answer.py` — IA com LangChain

Usa `gpt-4o-mini` via LangChain para classificar e responder comentários de alunos.

**Fluxo:**

```
comentário do aluno
      │
classification_template → model → StrOutputParser
      │
      ▼
categoria: "dúvida sobre python" | "dúvida administrativa" | "dúvida geral"
      │
RunnableBranch
      │
      ├─ python_feedback_template  → resposta técnica detalhada
      ├─ course_feedback_template  → resposta sobre o curso/suporte
      └─ general_feedback_template → resposta geral
      │
      ▼
string com a resposta em português
```

Em caso de qualquer exceção, retorna `"ERRO"`.

---

### `features/bot_env/bot_env.py` — Configuração

Singleton que lê o `.env` na inicialização. Expõe: `token`, `server_id`, `channel_id`, `open_api_key`, `bot_dev`. Se alguma variável obrigatória faltar ou tiver tipo inválido, chama `SystemExit`.

---

### `utils/singleton.py` — Padrão Singleton

Metaclasse usada por `BotEnv`, `BotLogger`, `Help` e `Teachers`. A instância é criada na primeira chamada à classe e reutilizada em todas as importações subsequentes. Testes precisam chamar `._clear()` explicitamente entre os casos para resetar o estado.

---

### `utils/worker.py` — Ponte sync → async

Permite rodar código bloqueante (chamadas ao Google Sheets, carregamento de parquet) em uma thread separada sem travar o event loop do asyncio.

```python
worker = Worker(blocking_function, *args)
worker.start()
data, error = await worker.get_result()
```

A thread chama a função e coloca o resultado em uma `Queue`. `get_result()` faz polling com `await asyncio.sleep()` até o resultado aparecer.

---

## Fluxo de dados dos comentários (passo a passo)

```
WordPress REST API
      │
      │  fetch_comments() — py_wordpress_api
      ▼
DataFrame bruto (post_id, author, message em HTML, ...)
      │
      │  preprocess_wordpress_comments()
      │    → parse HTML (BeautifulSoup)
      │    → resolve post_id → nome do curso (load_activities())
      │    → converte timestamps para GMT
      ▼
DataFrame normalizado
      │
      │  combine_comments()
      │    → concat com parquet local
      │    → linhas existentes nunca são sobrescritas
      ▼
DataFrame mesclado
      │
      │  assign_teachers()
      │    → Levenshtein match: nome do curso → professores
      │    → mantém professor da thread se já existe
      ▼
      │  assign_comment_status()
      │    → answer_pending=True se sem resposta de professor
      ▼
Parquet salvo em disco
      │
      │  get_comments_to_post() — filtra não postados
      ▼
      │  BotAI.answer(comentário) — LangChain
      ▼
      │  post_comments() — Discord
      ▼
Mensagem no canal Discord com:
  - @menção ao professor
  - dados do comentário
  - resposta rascunho da IA
```

---

## Painel de controle (Streamlit)

`dash/app.py` é uma interface web local para:
- Iniciar e parar o processo do bot
- Visualizar o log em tempo real (`~/.academy_support_bot/bot.log`)

`dash/run_bot_forever.py` lança o bot como subprocesso e reinicia a cada ciclo.

---

## Testes

Os testes cobrem apenas `features/teacher/teacher.py` (`tests/test_features/test_teacher.py`). O fixture `fake_teachers` em `conftest.py` substitui a chamada ao Google Sheets por objetos simulados e limpa o estado do Singleton antes de cada teste com `Teachers()._clear()`.

```bash
poetry run pytest                                                          # todos os testes
poetry run pytest tests/test_features/test_teacher.py::test_get_all_records  # teste específico
```

---

## Configuração e execução

```bash
# Instalar dependências
poetry install

# Rodar o bot
poetry run python -m academy_support_bot.bot

# Empacotar para produção (gera .pex em dist/)
poetry pack

# Limpar dist/
poetry clear
```

O arquivo `academy-support-bot-66332ff89041.json` (service account do Google) deve estar na pasta home do usuário (`~/`).
