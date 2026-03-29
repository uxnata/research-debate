# 🔬 Research Debate Lab

Мультиагентные дебаты для продуктовых исследований.

Три ИИ-аналитика анализируют данные через разные линзы, спорят между собой, а человек-исследователь управляет дебатами и принимает решения.

> Вдохновлено [ARIS](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep) — adversarial collaboration паттерн, адаптированный из ML-исследований для UX-ресёрча.

## Зачем

Один LLM-чат с транскриптами даёт плоский отчёт. Мультиагентные дебаты создают **конфликт интерпретаций**, который выявляет слепые зоны и ложные выводы.

## Архитектура

```
Данные → [Адвокат пользователя] + [Адвокат бизнеса] → [Критик] → Человек решает → Отчёт
```

- **Адвокат пользователя** — эмоции, JTBD, потерянные привычки
- **Адвокат бизнеса** — LTV, сегменты, unit-экономика
- **Критик** — атакует обоих, ищет bias и слепые зоны
- **Человек** — направляет дебаты, добавляет контекст, принимает решения

## Быстрый старт

1. Склонируйте / скопируйте в рабочую директорию
2. Положите свои данные в `input/transcripts.md` и `input/context.md`
3. Откройте в Claude Code
4. Запустите: `/research-debate`

## Структура

```
research-debate-skills/
├── CLAUDE.md                              # конфиг проекта
├── README.md                              # этот файл
├── .mcp.json                              # конфиг MCP серверов
├── input/
│   ├── transcripts.md                     # ваши данные
│   └── context.md                         # контекст исследования
├── output/                                # результаты
└── skills/
    ├── research-debate/SKILL.md           # оркестратор
    ├── user-advocate/SKILL.md             # Agent A
    ├── biz-advocate/SKILL.md              # Agent B
    ├── critic/SKILL.md                    # Agent C (self-review)
    ├── cross-model-critic/SKILL.md        # Agent C (через MCP)
    └── synthesis/SKILL.md                 # финальный отчёт
```

## Cross-model режим

Для настоящего adversarial review — Критик должен быть другой моделью.

### Быстрая настройка (выберите один вариант)

**Codex CLI (GPT-5, бесплатно с ChatGPT):**
```bash
npm install -g @openai/codex
claude mcp add -s user codex-cli -- npx -y codex-cli-mcp-tool
# Установите OPENAI_API_KEY в окружении
```

**OpenRouter (любая модель, включая open source):**
```bash
claude mcp add -s user openrouter -- npx -y openrouter-mcp
# Установите OPENROUTER_API_KEY в окружении
```

**Gemini:**
```bash
npm install -g gemini-mcp-tool
claude mcp add -s user gemini-cli -- npx -y gemini-mcp-tool
# Аутентификация через Google аккаунт
```

В `CLAUDE.md` уже стоит `CROSS_MODEL_CRITIC: true`. Оркестратор автоматически направит раунд критики через MCP.

### Почему это важно

| Self-review (Claude → Claude) | Cross-model (Claude → GPT-5) |
|---|---|
| Одинаковые training biases | Разные biases |
| Склонность соглашаться с собой | Нет loyalty к предыдущим выводам |
| Одинаковый стиль рассуждений | Другой подход к логике |

Claude Code анализирует (A+B), внешняя модель критикует (C). Разные модели = разные слепые зоны = лучший результат.

## Выходные документы

- `RESEARCH_REPORT.md` — финальный отчёт с консенсусными, скорректированными и опровергнутыми выводами
- `DIVERGENCE_MAP.md` — где агенты не согласны (зоны для дополнительного исследования)
- `ASSUMPTIONS_LOG.md` — все допущения агентов (потенциальные риски)

## Совместимость

Чистый Markdown, ноль зависимостей. Работает с:
- Claude Code
- Codex CLI
- OpenClaw
- Cursor (через @file)
- Любой LLM-агент, читающий SKILL.md

## Лицензия

MIT
