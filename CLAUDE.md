# Research Debate Lab

Мультиагентная система для продуктовых исследований. Три аналитика с разными линзами спорят о данных, человек-исследователь управляет дебатами.

## Скиллы

Скиллы находятся в `skills/`. Каждый — директория с `SKILL.md`.

- `/research-debate` — полный пайплайн (оркестратор)
- `/user-advocate` — анализ через линзу пользовательского опыта
- `/biz-advocate` — анализ через линзу бизнес-метрик
- `/critic` — adversarial критика выводов A и B (self-review)
- `/cross-model-critic` — критика через внешнюю модель (рекомендуется)
- `/evidence-verifier` — проверка цитат в findings против исходных транскриптов
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
CRITIC_MODEL: openai/gpt-4o       # модель OpenRouter для критики
OPENROUTER_API_KEY: env            # ключ передаётся через переменную окружения $OPENROUTER_API_KEY
AUTO_PROCEED: false                # true = без чекпоинтов
MAX_ROUNDS: 3                      # максимум раундов дебатов
LANGUAGE: ru                       # язык вывода

## Cross-Model Critic

Для cross-model критики используется OpenRouter API через curl (без MCP).

**Настройка:**
```bash
export OPENROUTER_API_KEY="ваш-ключ"
```

**Поддерживаемые модели** (задаются через `CRITIC_MODEL`):
- `openai/gpt-4o` — хороший баланс (по умолчанию)
- `google/gemini-2.0-flash-thinking` — сильный в логике
- `meta-llama/llama-3.3-70b` — open source альтернатива

## Правила

- Все выходные файлы записываются в `output/`
- JSON-выходы агентов должны быть валидным JSON
- Каждый finding обязан содержать evidence (цитату или данные)
- При AUTO_PROCEED: false — останавливайся на чекпоинте и спрашивай человека
- Не модифицируй файлы в `input/` и `skills/`
