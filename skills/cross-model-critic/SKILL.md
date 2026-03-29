# Cross-Model Critic Bridge

Мост для выполнения критики через внешнюю модель. Устраняет проблему self-review blind spots: Claude Code никогда не ревьюит свои собственные выводы.

> **Принцип ARIS:** скорость × строгость. Claude Code (executor) анализирует быстро и гибко. Внешняя модель (reviewer) критикует медленно и строго. Разные модели = разные слепые зоны = лучший результат.

## Когда использовать

Вместо `/critic` — когда `CROSS_MODEL_CRITIC: true` в CLAUDE.md.

## Поддерживаемые варианты

### Вариант 1: Codex CLI MCP (рекомендуется)

Требования:
- ChatGPT подписка (даёт бесплатный доступ к GPT-5 через Codex CLI)
- Установлен `codex-cli-mcp-tool`

```bash
# Установка
npm install -g @openai/codex
claude mcp add -s user codex-cli -- npx -y codex-cli-mcp-tool
```

Как работает: Claude Code собирает контекст → отправляет через MCP → Codex (GPT-5) критикует → результат возвращается в Claude Code.

### Вариант 2: OpenRouter MCP

Требования:
- API ключ OpenRouter
- Установлен `openrouter-mcp`

```bash
claude mcp add -s user openrouter -- npx -y openrouter-mcp
```

Модели для критики через OpenRouter:
- `openai/gpt-4o` — хороший баланс
- `google/gemini-2.0-flash-thinking` — сильный в логике
- `meta-llama/llama-3.3-70b` — open source альтернатива

### Вариант 3: Gemini MCP

```bash
npm install -g gemini-mcp-tool
claude mcp add -s user gemini-cli -- npx -y gemini-mcp-tool
```

## Протокол выполнения

### Шаг 1: Подготовь контекст

Прочитай и собери в один текст:
- `input/transcripts.md`
- `input/context.md`
- `output/round1-user-advocate.json`
- `output/round1-biz-advocate.json`

### Шаг 2: Собери промпт для внешней модели

```
Ты — Критик в мультиагентной исследовательской системе.
Ты получаешь выводы двух аналитиков и должен АТАКОВАТЬ их.

## Правила
- Бери КОНКРЕТНЫЙ finding по ID и атакуй его
- Не говори «может быть неверно» — говори «неверно, ПОТОМУ ЧТО...»
- Предлагай КОНКРЕТНУЮ альтернативную интерпретацию
- Если вывод железобетонный — скажи это (strongest_findings)
- Особенно ищи случаи, когда ОБА аналитика согласны

## Исходные данные исследования
{содержимое input/transcripts.md}

## Контекст проекта
{содержимое input/context.md}

## Выводы Адвоката пользователя
{содержимое output/round1-user-advocate.json}

## Выводы Адвоката бизнеса
{содержимое output/round1-biz-advocate.json}

## Формат ответа
Ответь СТРОГО в JSON:
{
  "agent": "critic",
  "round": 2,
  "model_used": "[указать модель]",
  "challenges": [
    {
      "id": "CR-1",
      "target": "UA-1 или BA-1",
      "target_claim": "цитата атакуемого утверждения",
      "attack_type": "confirmation_bias | weak_evidence | false_causality | missed_alternative | blind_spot | agreement_bias",
      "attack": "почему неверно",
      "alternative_explanation": "альтернативная интерпретация",
      "severity": "critical | moderate | minor"
    }
  ],
  "blind_spots": [
    {
      "id": "CR-BS1",
      "description": "что ОБА проигнорировали",
      "potential_significance": "почему важно"
    }
  ],
  "strongest_findings": ["UA-X", "BA-Y"],
  "meta_observation": "главная слабость обоих"
}
```

### Шаг 3: Отправь через MCP

**Codex CLI:**
```
Используй инструмент codex-cli для отправки следующего промпта.
Модель: gpt-5. Режим: read-only.
[промпт из шага 2]
```

**OpenRouter:**
```
Используй инструмент openrouter для отправки запроса.
Модель: openai/gpt-4o (или другая из списка).
[промпт из шага 2]
```

**Gemini:**
```
Используй инструмент gemini-cli для отправки запроса.
[промпт из шага 2]
```

### Шаг 4: Сохрани результат

1. Парси JSON из ответа внешней модели (может быть обёрнут в ```json```)
2. Добавь поле `"cross_model": true` и `"model_used": "gpt-5"` (или какая использована)
3. Запиши в `output/round2-critic.json`

### Шаг 5: Выведи сводку

```
═══════════════════════════════════════
  КРИТИКА (cross-model: GPT-5)
═══════════════════════════════════════

  Challenges: N штук
    critical: X | moderate: Y | minor: Z

  Слепые зоны: N

  Устоявшие выводы: [список ID]

  Главное наблюдение: "..."
═══════════════════════════════════════
```

## Fallback

Если MCP недоступен или модель не отвечает:
1. Выведи предупреждение: "Cross-model MCP недоступен, использую self-review (менее надёжно)"
2. Выполни `/critic` стандартным способом (Claude критикует свои выводы)
3. Добавь `"cross_model": false, "fallback_reason": "MCP unavailable"` в JSON

## Почему cross-model лучше

| Self-review (Claude→Claude) | Cross-model (Claude→GPT-5) |
|---|---|
| Одинаковые training biases | Разные biases |
| Одинаковые слепые зоны | Разные слепые зоны |
| Склонность соглашаться с собой | Нет loyalty к предыдущим выводам |
| Одинаковый стиль рассуждений | Другой подход к логике |
