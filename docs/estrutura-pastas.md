# Estrutura de pastas

[← Voltar ao README](../README.md)

```
academy_support_bot/
│
├── bot.py                          # Ponto de entrada — define SupportBot e main()
├── comment_handler.py              # Gerenciador do banco de dados de comentários (parquet)
├── comment_processing.py           # Pipeline ETL: busca, normaliza, mescla e classifica comentários
├── async_functions.py              # Formata e envia mensagens no Discord
│
├── cogs/                           # Camada Discord — registra slash commands e tasks
│   ├── comments_cog.py             # Task periódica (12 min) + /my_comments, /pending_comments, /dump_pending, /close_comment
│   ├── teachers_cog.py             # /teachers, /load_spreadsheet, /question
│   ├── help_cog.py                 # /help
│   └── uptime_cog.py               # /uptime
│
├── features/                       # Lógica de negócio isolada por domínio
│   ├── bot_ai/
│   │   └── comment_answer.py       # Classificador + gerador de respostas via LangChain (GPT-4o-mini)
│   ├── bot_comments/
│   │   └── bot_comments.py         # Orquestrador do fluxo de comentários (BotComments)
│   ├── bot_env/
│   │   └── bot_env.py              # Singleton: carrega e valida variáveis de ambiente
│   ├── bot_logger/
│   │   └── bot_logger.py           # Singleton: log rotativo em arquivo (~/.academy_support_bot/bot.log)
│   ├── bot_pidfile/
│   │   └── bot_pidfile.py          # Guarda de instância única via PID file
│   ├── chatgpt/
│   │   └── chatgpt.py              # Cliente OpenAI direto com saída estruturada (Pydantic)
│   ├── help/
│   │   └── help.py                 # Singleton: agrega descrições de comandos para o /help
│   ├── simple_doc/
│   │   └── simple_doc.py           # Gera arquivos .docx a partir de respostas da IA
│   ├── teacher/
│   │   ├── teacher.py              # Dataclass Teacher + Singleton Teachers (Google Sheets)
│   │   └── util.py                 # Type aliases e helpers de validação
│   └── uptime/
│       └── uptime.py               # Calcula tempo de execução do processo
│
├── utils/                          # Utilitários genéricos
│   ├── singleton.py                # Metaclasse Singleton usada em features/
│   ├── worker.py                   # Ponte entre código bloqueante e asyncio (Queue + Thread)
│   ├── auxiliary_functions.py      # Helpers de string e print
│   └── messages.py                 # emit_message() com timestamp
│
├── legacy/                         # Implementação antiga — não utilizada em produção
│   ├── run_bot.py                  # Bot original baseado em discord.Client (sem cogs)
│   ├── bot_client.py               # Versão legada do cliente Discord
│   ├── teachers.py                 # Tabela de professores hardcoded (substituída pelo Google Sheets)
│   └── content.py                  # Tabela de cursos hardcoded (substituída pelo Google Sheets)
│
├── dash/                           # Painel de controle Streamlit
│   ├── app.py                      # Interface web: iniciar/parar bot, visualizar log
│   └── run_bot_forever.py          # Lançador de processo com loop de reinício
│
├── analysis/
│   └── comments_analysis.py        # Scripts de análise offline do banco de comentários (matplotlib)
│
└── comment_migration_app/
    └── app.py                      # Ferramenta Streamlit para inspecionar backup histórico de comentários
```

---

## Onde ficam os dados em tempo de execução

| Arquivo | Caminho | Conteúdo |
|---|---|---|
| Banco de comentários | `~/.academy_support_bot/bot.db` | DataFrame parquet com todos os comentários e seu status |
| Log do bot | `~/.academy_support_bot/bot.log` | Log rotativo (máx. 4 KB × 10 arquivos) |
| PID file | `~/.academy_support_bot/bot.pid` | PID do processo ativo (evita instâncias duplicadas) |
| Credencial Google | `~/academy-support-bot-66332ff89041.json` | Service account para leitura do Google Sheets |

---

## Onde fica o código de testes

```
tests/
├── conftest.py                     # Fixtures: objetos falsos de planilha Google + reset do Singleton
└── test_features/
    └── test_teacher.py             # Testes de Teachers (carregamento, atribuição, busca)
```

Os testes cobrem apenas `features/teacher/teacher.py`. As chamadas ao Google Sheets são substituídas por objetos simulados (`fake_teacher_spreadsheet`, `fake_content_spreadsheet`) definidos em `conftest.py`.
