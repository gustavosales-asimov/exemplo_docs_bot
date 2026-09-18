# Academy Support Bot

Bot de Discord da Asimov Academy que automatiza o suporte a alunos. Ele busca perguntas deixadas nos comentários das aulas na plataforma WordPress, distribui cada comentário para um professor responsável, posta as pendências no Discord em intervalos regulares e gera uma resposta rascunho com IA para cada dúvida.

---

## Como funciona (visão geral)

```
WordPress REST API
      │  busca comentários a cada 12 min
      ▼
  Pipeline ETL (comment_processing.py)
      │  normaliza, mescla com DB local,
      │  atribui professor, classifica status
      ▼
  Banco local (parquet)
      │
      ├─ BotAI (LangChain + GPT-4o-mini)
      │    classifica dúvida → gera rascunho de resposta
      │
      └─ Discord (async_functions.py)
           posta comentário + rascunho no canal,
           menciona o professor responsável
```

Os professores recebem a notificação no Discord com a dúvida do aluno e um rascunho de resposta gerado pela IA. Eles também podem usar slash commands para gerenciar sua fila.

---

## Slash commands disponíveis

| Comando | Descrição |
|---|---|
| `/my_comments` | Lista as pendências atribuídas ao professor que chamou |
| `/pending_comments` | Resumo de todas as pendências abertas |
| `/dump_pending` | Lista completa de todas as pendências |
| `/close_comment <id>` | Fecha um comentário manualmente |
| `/teachers` | Lista os professores ativos |
| `/load_spreadsheet` | Recarrega os dados de professores do Google Sheets |
| `/question` | Menciona um professor aleatório para questões avulsas |
| `/help` | Lista todos os comandos disponíveis |
| `/uptime` | Exibe o tempo de execução do bot |

---

## Configuração

### Pré-requisitos

- Python `>=3.11,<3.12`
- [Poetry](https://python-poetry.org/)
- Arquivo de credenciais do Google (`academy-support-bot-66332ff89041.json`) na pasta home (`~/`)

### Variáveis de ambiente

Crie um arquivo `.env` na raiz do projeto:

```env
DISCORD_TOKEN=...
DISCORD_SERVER_ID=...
DISCORD_CHANNEL_ID=...
OPENAI_API_KEY=...
WP_USERNAME=...
WP_PASSWORD=...
BOT_DEV=false        # true para usar a planilha de teste do Google Sheets
```

### Instalação e execução

```bash
# Instalar dependências
poetry install

# Rodar o bot
poetry run python -m academy_support_bot.bot

# Rodar os testes
poetry run pytest

# Empacotar para produção (gera .pex em dist/)
poetry pack

# Limpar dist/
poetry clear
```

---

## Documentação detalhada

- [Estrutura de pastas](docs/estrutura-pastas.md) — mapa comentado de todos os arquivos do projeto
- [Módulos principais](docs/modulos-principais.md) — como cada parte funciona e como se conectam
