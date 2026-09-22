# AwrAIter Agent Skills

Скилл для AI-агентов (Claude Code, Codex и любой агент со скиллами) для работы с [AwrAIter](https://awraiter.ai): Telegram-каналы, которыми вы управляете — аналитика, черновики из собственных источников канала, контент-планы, публикация по расписанию — через [MCP-сервер](https://awraiter.ai/mcp-server/) или REST v1. Без юзербота: ни номера телефона, ни кода входа.

[English](README.md) · [SKILL.md](skills/awraiter/SKILL.md)

## Установка

Claude Code:

```bash
claude plugin marketplace add MediaZione/AwrAiter_skills
claude plugin install awraiter@awraiter
```

Другие агенты:

```bash
npx skills add MediaZione/AwrAiter_skills --skill awraiter
```

Вручную: скопируйте `skills/awraiter/SKILL.md` в папку скиллов агента (для Claude Code — `~/.claude/skills/awraiter/SKILL.md`).

## Подключение

- **MCP с OAuth** (Claude, Claude Code, chatgpt.com): добавьте `https://awraiter.ai/mcp` — клиент сам проведёт вход через Telegram.
- **Персональный токен** (REST v1 и MCP-клиенты без OAuth): создайте в панели, раздел [MCP-подключение](https://awraiter.ai/mcp-connect), и экспортируйте:

```bash
export AWRAITER_TOKEN=pat_...
```

Пошагово для каждого клиента: [docs/connect.ru.md](https://github.com/MediaZione/AwrAiter/blob/main/docs/connect.ru.md).

## Использование

Попросите агента, например:

> Покажи статистику моего канала за неделю и подготовь два поста из свежих источников; перед постановкой в расписание покажи превью.

В Claude Code скилл вызывается явно: `/awraiter:awraiter`.

## Лицензия

[MIT](LICENSE).
