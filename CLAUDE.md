# Research Debate Lab

Мультиагентная система для продуктовых исследований. Три аналитика с разными линзами спорят о данных, человек-исследователь управляет дебатами.

## Скиллы

Скиллы находятся в `skills/`. Каждый — директория с `SKILL.md`.

- `/research-debate` — полный пайплайн (оркестратор)
- `/user-advocate` — анализ через линзу пользовательского опыта
- `/biz-advocate` — анализ через линзу бизнес-метрик
- `/critic` — adversarial критика выводов A и B (self-review)
- `/cross-model-critic` — критика через внешнюю модель (рекомендуется)
- `/synthesis` — финальный отчёт, карта расхождений, лог допущений

## Быстрый старт

```
/research-debate
```

Это запустит полный пайплайн: чтение данных → анализ A и B → критика C → чекпоинт → синтез.

## Данные

- `input/transcripts.md` — транскрипты интервью
- `input/context.md` — контекст проекта и метрики
- `output/` — результаты (создаётся автоматически)

## Конфигурация

CROSS_MODEL_CRITIC: true           # Критик через внешнюю модель (рекомендуется)
CRITIC_MCP: codex-cli              # codex-cli | openrouter | gemini-cli
AUTO_PROCEED: false                # true = без чекпоинтов
MAX_ROUNDS: 3                      # максимум раундов дебатов
LANGUAGE: ru                       # язык вывода

## MCP серверы

Настроены в `.mcp.json`. Для cross-model критики нужен хотя бы один:

### Вариант A: Codex CLI (GPT-5, бесплатно с ChatGPT подпиской)
```bash
npm install -g @openai/codex
claude mcp add -s user codex-cli -- npx -y codex-cli-mcp-tool
```
Переменная окружения: `OPENAI_API_KEY`

### Вариант B: OpenRouter (любая модель)
```bash
claude mcp add -s user openrouter -- npx -y openrouter-mcp
```
Переменная окружения: `OPENROUTER_API_KEY`

### Вариант C: Gemini
```bash
npm install -g gemini-mcp-tool
claude mcp add -s user gemini-cli -- npx -y gemini-mcp-tool
```
Аутентификация через Google аккаунт.

## Правила

- Все выходные файлы записываются в `output/`
- JSON-выходы агентов должны быть валидным JSON
- Каждый finding обязан содержать evidence (цитату или данные)
- При AUTO_PROCEED: false — останавливайся на чекпоинте и спрашивай человека
- Не модифицируй файлы в `input/` и `skills/`
