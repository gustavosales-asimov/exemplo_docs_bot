# Módulos principais

[← Voltar ao README](../README.md)

---

## Sumário

- [bot.py — Ponto de entrada](#botpy--ponto-de-entrada)
- [Pipeline de comentários](#pipeline-de-comentários)
  - [comment_processing.py — ETL](#comment_processingpy--etl)
  - [comment_handler.py — Banco de dados](#comment_handlerpy--banco-de-dados)
  - [async_functions.py — Envio no Discord](#async_functionspy--envio-no-discord)
- [Cogs (camada Discord)](#cogs-camada-discord)
- [features/teacher — Modelo de professores](#featuresteacher--modelo-de-professores)
- [features/bot_comments — Orquestrador](#featuresbot_comments--orquestrador)
- [features/bot_ai — IA com LangChain](#featuresbot_ai--ia-com-langchain)
- [Utilitários transversais](#utilitários-transversais)
- [Fluxo de dados completo](#fluxo-de-dados-completo)

---

## `bot.py` — Ponto de entrada

Define `SupportBot(commands.Bot)` e a função `main()` que inicializa o processo.

**Inicialização:**

1. `BotEnv` carrega e valida as variáveis de ambiente (falha com `SystemExit` se algo estiver errado)
2. `is_single_instance()` verifica o PID file — impede dois processos rodando ao mesmo tempo
3. `setup_hook()` carrega os quatro cogs em sequência (com 1 s de intervalo) e sincroniza os slash commands com o servidor Discord configurado

Os comandos são sincronizados apenas para a guild configurada em `DISCORD_SERVER_ID`, não globalmente — isso torna as atualizações instantâneas durante o desenvolvimento.

---

## Pipeline de comentários

O fluxo central do bot é disparado pelo `CommentsCog` a cada 12 minutos e percorre três módulos em sequência:

```
comment_processing.py  →  comment_handler.py  →  async_functions.py
   (busca e ETL)            (banco de dados)        (envio no Discord)
```

### `comment_processing.py` — ETL

É o coração do bot. Toda execução de `load_comments()` percorre estas etapas:

```
1. fetch_wordpress_comments()       ← py_wordpress_api (REST API do WordPress)
        │
2. preprocess_wordpress_comments()  ← normaliza colunas, parseia HTML (BeautifulSoup),
        │                              resolve post_id → nome do curso via load_activities()
        │
3. load_local_comments()            ← lê o parquet local (~/.academy_support_bot/bot.db)
        │
4. combine_comments()               ← concatena novos comentários; nunca sobrescreve os existentes
        │                              (preserva fechamentos manuais e respostas da IA já salvas)
        │
5. assign_teachers()                ← distribui comentários para professores
        │
6. assign_comment_status()          ← determina quais comentários estão pendentes
        │
7. Salva no parquet                 ← persiste o estado atualizado em disco
```

**`assign_teachers()`**

Para cada comentário sem professor atribuído, busca os professores que cobrem aquele curso via `Teachers.get_teachers_by_content()`. A correspondência usa distância de Levenshtein com tolerância de até 4 caracteres — o que permite encontrar o professor mesmo com pequenas variações no nome do curso.

Se um professor já respondeu em uma thread, todos os outros comentários da mesma thread recebem automaticamente o mesmo professor.

**`assign_comment_status()`**

Um comentário recebe `answer_pending = True` quando:
- Foi feito por um aluno (o `author_id` não está na lista de IDs de professores no WordPress)
- Não tem nenhuma resposta de professor ainda
- Não foi fechado manualmente

Se o último comentário de uma thread for de um professor, toda a thread é marcada como resolvida (`answer_pending = False`).

---

### `comment_handler.py` — Banco de dados

Gerencia o DataFrame pandas em memória e o arquivo parquet em disco. É a fonte de verdade do estado dos comentários durante a sessão.

| Método | O que faz |
|---|---|
| `reload_comments()` | Roda `load_comments()` em thread separada; atualiza `comments_df` |
| `get_comments_to_post()` | Retorna pendentes ainda não postados nesta sessão |
| `mark_as_posted(ids)` | Adiciona IDs ao conjunto de já postados (apenas em memória) |
| `close_comment(id)` | Fecha o comentário e toda a thread; salva no parquet |
| `save_bot_comment(id, texto)` | Salva a resposta gerada pela IA; salva no parquet |

O conjunto `posted_comment_ids` vive apenas em memória — ao reiniciar o bot, todos os comentários pendentes voltam a ser elegíveis para postagem.

---

### `async_functions.py` — Envio no Discord

`post_comments(comments_df, channel)` formata cada linha do DataFrame como uma mensagem Markdown e envia ao canal configurado. A mensagem inclui:

- Menção ao professor responsável (`<@discord_id>`)
- ID do comentário, timestamp, nome do curso e nome do aluno
- Prévia da mensagem do aluno (máximo 1 400 caracteres)
- Rascunho de resposta da IA como segunda mensagem, se disponível

Entre cada envio há um `await asyncio.sleep(2)` para respeitar o rate limit do Discord.

`post_comment_summary()` envia uma única mensagem de resumo com o total de pendências e a distribuição por professor e por curso.

---

## Cogs (camada Discord)

Os cogs são a interface entre o Discord e a lógica de negócio. Cada um é uma classe fina que registra comandos e delega para uma feature:

| Cog | Feature | Responsabilidade |
|---|---|---|
| `comments_cog.py` | `BotComments` | Task de 12 min + comandos de comentários |
| `teachers_cog.py` | `Teachers` | Carrega planilha na inicialização + comandos de professores |
| `help_cog.py` | `Help` | Registra e lista todos os comandos |
| `uptime_cog.py` | `uptime()` | Exibe tempo de execução |

`TeachersCog.cog_load()` é o único lugar onde uma falha é fatal: se `Teachers.load_spreadsheet()` lançar exceção, o processo inteiro encerra — sem professores carregados o bot não tem como funcionar.

---

## `features/teacher` — Modelo de professores

### `Teacher` (dataclass)

| Campo | Tipo | Descrição |
|---|---|---|
| `name` | `str` | Nome do professor |
| `wp_id` | `int` | ID no WordPress (para identificar respostas de professores em comentários) |
| `discord_id` | `int` | ID no Discord (para a menção `<@id>`) |
| `contents` | `list[str]` | Nomes dos cursos/projetos atribuídos |
| `question` | `bool` | Disponível para questões avulsas via `/question` |

`is_content_assigned(content, exact=False)` compara `content` com cada item de `contents` via distância de Levenshtein. Com `exact=False`, aceita até 4 caracteres de diferença.

### `Teachers` (Singleton)

Carregado uma vez na inicialização via `load_spreadsheet()`. Lê quatro abas do Google Sheets:

- `teachers` — lista de professores ativos com seus IDs e configurações
- `previous_teachers` — professores inativos (usados para reconhecer respostas antigas de professores)
- `existing_courses` — matriz `curso × professor` (coluna com `"x"` indica que o professor cobre aquele curso)
- `existing_projects` — mesma estrutura para projetos

A autenticação usa o arquivo `~/academy-support-bot-66332ff89041.json` (service account do Google).

Como a chamada ao Google Sheets é bloqueante, ela roda em uma thread separada via `Worker`, evitando travar o event loop do asyncio.

---

## `features/bot_comments` — Orquestrador

`BotComments` é a camada intermediária entre os cogs e os módulos de baixo nível. Coordena o fluxo completo a cada ciclo de 12 minutos:

```python
# ordem de execução em post_available_comments_to_channel()
comment_handler.reload_comments()    # 1. busca novos comentários do WordPress
get_comments_to_post()               # 2. filtra os ainda não postados nesta sessão
_add_ai_comments()                   # 3. gera rascunhos com IA para os sem resposta
post_comments()                      # 4. envia ao canal Discord
```

`_add_ai_comments()` chama `BotAI.answer()` para cada comentário sem `bot_comment` e salva o resultado no parquet antes de postar, garantindo que rascunhos não sejam gerados duas vezes.

---

## `features/bot_ai` — IA com LangChain

`BotAI` usa `gpt-4o-mini` via LangChain para classificar e responder comentários de alunos em duas etapas:

```
comentário do aluno
      │
      ▼
classification_template → model → StrOutputParser
      │
      ▼  categoria: "python" | "administrativa" | "geral"
      │
RunnableBranch
      ├─ python_feedback_template   → resposta técnica detalhada
      ├─ course_feedback_template   → resposta sobre o curso ou suporte
      └─ general_feedback_template  → resposta geral
      │
      ▼
string com a resposta em português
```

Em caso de qualquer exceção, o método `answer()` retorna `"ERRO"` em vez de propagar a falha, para não interromper o ciclo de postagem.

> **Nota:** existe também `features/chatgpt/chatgpt.py` (`ChatGpt`), um cliente OpenAI alternativo que usa saída estruturada (Pydantic). Ele não está conectado ao fluxo principal de comentários — é usado apenas pela feature `SimpleDoc` para geração de `.docx`.

---

## Utilitários transversais

### `utils/singleton.py`

Metaclasse que garante uma única instância por classe. Usada por `BotEnv`, `BotLogger`, `Help` e `Teachers`. Em testes, o estado deve ser limpo explicitamente com `._clear()` entre os casos (veja `tests/conftest.py`).

### `utils/worker.py`

Permite rodar código bloqueante em uma thread separada sem travar o event loop do asyncio:

```python
worker = Worker(blocking_function, *args)
worker.start()
data, error = await worker.get_result()  # polling com asyncio.sleep() até o resultado
```

Usado em `Teachers.load_spreadsheet()` (Google Sheets) e `CommentHandler.reload_comments()` (WordPress API + parquet).

### `features/bot_env/bot_env.py`

Singleton que lê o `.env` na inicialização e expõe `token`, `server_id`, `channel_id`, `open_api_key` e `bot_dev`. Encerra o processo com `SystemExit` se qualquer variável obrigatória estiver ausente ou com tipo inválido.

### `features/bot_logger/bot_logger.py`

Singleton que envolve `logging` com um `RotatingFileHandler`. O arquivo de log fica em `~/.academy_support_bot/bot.log` (máximo 4 KB por arquivo, 10 arquivos de backup). O handler é passado ao `SupportBot.run()` para que os logs internos do discord.py também sejam redirecionados para o mesmo arquivo.

---

## Fluxo de dados completo

```
WordPress REST API
      │
      │  fetch_wordpress_comments() — py_wordpress_api
      ▼
DataFrame bruto
  (post_id, author_id, message em HTML, datetime, ...)
      │
      │  preprocess_wordpress_comments()
      │    → BeautifulSoup: remove tags HTML
      │    → load_activities(): resolve post_id → nome e tipo do curso
      │    → converte timestamps para GMT
      ▼
DataFrame normalizado
      │
      │  combine_comments()
      │    → concat com parquet local
      │    → registros existentes nunca são sobrescritos
      ▼
DataFrame mesclado
      │
      │  assign_teachers()
      │    → Levenshtein match: nome do curso → lista de professores
      │    → propaga professor da thread para comentários relacionados
      ▼
      │  assign_comment_status()
      │    → answer_pending=True: autor é aluno e sem resposta de professor
      │    → answer_pending=False: último comentário da thread é de professor
      ▼
Salvo no parquet (~/.academy_support_bot/bot.db)
      │
      │  get_comments_to_post()
      │    → filtra: answer_pending=True e não postado nesta sessão
      ▼
      │  BotAI.answer(comentário) — LangChain + gpt-4o-mini
      │    → classifica → gera rascunho em português
      │    → save_bot_comment() → persiste no parquet
      ▼
      │  post_comments() — async_functions.py
      │    → formata mensagem Markdown por comentário
      │    → menciona professor (<@discord_id>)
      │    → envia comentário + rascunho ao canal Discord
      ▼
Canal Discord
  - Mensagem 1: dados do comentário + menção ao professor
  - Mensagem 2: rascunho de resposta da IA (se disponível)
```
