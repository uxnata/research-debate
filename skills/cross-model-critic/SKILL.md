# Cross-Model Critic Bridge

Мост для выполнения критики через внешнюю модель. Устраняет проблему self-review blind spots: Claude Code никогда не ревьюит свои собственные выводы.

> **Принцип ARIS:** скорость × строгость. Claude Code (executor) анализирует быстро и гибко. Внешняя модель (reviewer) критикует медленно и строго. Разные модели = разные слепые зоны = лучший результат.

## Когда использовать

Вместо `/critic` — когда `CROSS_MODEL_CRITIC: true` в CLAUDE.md.

## Как работает

Claude Code собирает контекст → вызывает OpenRouter API через curl → внешняя модель критикует → результат парсится и сохраняется.

**Требования:**
- Переменная окружения `OPENROUTER_API_KEY`

**Модели для критики:**
- `openai/gpt-4o` — хороший баланс (по умолчанию)
- `google/gemini-2.0-flash-thinking` — сильный в логике
- `meta-llama/llama-3.3-70b` — open source альтернатива

Модель задаётся через `CRITIC_MODEL` в CLAUDE.md.

## Протокол выполнения

### Шаг 1: Подготовь контекст

Прочитай и собери в переменные:
- `input/transcripts.md`
- `input/context.md`
- `output/round1-user-advocate.json`
- `output/round1-biz-advocate.json`

### Шаг 2: Собери промпт для внешней модели

**System prompt:**

```
Ты — Критик в мультиагентной исследовательской системе.
Ты получаешь выводы двух аналитиков и должен АТАКОВАТЬ их.

## Правила
- Бери КОНКРЕТНЫЙ finding по ID и атакуй его
- Не говори «может быть неверно» — говори «неверно, ПОТОМУ ЧТО...»
- Предлагай КОНКРЕТНУЮ альтернативную интерпретацию
- Если вывод железобетонный — скажи это (strongest_findings)
- Особенно ищи случаи, когда ОБА аналитика согласны

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

**User prompt:**

```
## Исходные данные исследования
{содержимое input/transcripts.md}

## Контекст проекта
{содержимое input/context.md}

## Выводы Адвоката пользователя
{содержимое output/round1-user-advocate.json}

## Выводы Адвоката бизнеса
{содержимое output/round1-biz-advocate.json}
```

### Шаг 3: Вызови OpenRouter через curl

```bash
CRITIC_RESPONSE=$(curl -s https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-4o",
    "messages": [
      {"role": "system", "content": "[system prompt из шага 2]"},
      {"role": "user", "content": "[user prompt из шага 2]"}
    ],
    "temperature": 0.7,
    "max_tokens": 4000
  }')
```

> **Важно:** JSON в теле запроса должен экранировать спецсимволы из промптов. Используй `jq` для безопасной сборки payload:
>
> ```bash
> PAYLOAD=$(jq -n \
>   --arg model "openai/gpt-4o" \
>   --arg system "$SYSTEM_PROMPT" \
>   --arg user "$USER_PROMPT" \
>   '{
>     model: $model,
>     messages: [
>       {role: "system", content: $system},
>       {role: "user", content: $user}
>     ],
>     temperature: 0.7,
>     max_tokens: 4000
>   }')
>
> CRITIC_RESPONSE=$(curl -s https://openrouter.ai/api/v1/chat/completions \
>   -H "Authorization: Bearer $OPENROUTER_API_KEY" \
>   -H "Content-Type: application/json" \
>   -d "$PAYLOAD")
> ```

### Шаг 4: Парси и сохрани результат

1. Извлеки content из ответа: `echo "$CRITIC_RESPONSE" | jq -r '.choices[0].message.content'`
2. Парси JSON (может быть обёрнут в ```json```)
3. Добавь поля `"cross_model": true` и `"model_used": "openai/gpt-4o"` (или какая использована)
4. Запиши в `output/round2-critic.json`

### Шаг 5: Выведи сводку

```
═══════════════════════════════════════
  КРИТИКА (cross-model: openai/gpt-4o)
═══════════════════════════════════════

  Challenges: N штук
    critical: X | moderate: Y | minor: Z

  Слепые зоны: N

  Устоявшие выводы: [список ID]

  Главное наблюдение: "..."
═══════════════════════════════════════
```

## Fallback

Если `OPENROUTER_API_KEY` не задан или curl возвращает ошибку:
1. Выведи предупреждение: "OpenRouter API недоступен, использую self-review (менее надёжно)"
2. Выполни `/critic` стандартным способом (Claude критикует свои выводы)
3. Добавь `"cross_model": false, "fallback_reason": "OpenRouter API unavailable"` в JSON

## Почему cross-model лучше

| Self-review (Claude→Claude) | Cross-model (Claude→GPT-4o) |
|---|---|
| Одинаковые training biases | Разные biases |
| Одинаковые слепые зоны | Разные слепые зоны |
| Склонность соглашаться с собой | Нет loyalty к предыдущим выводам |
| Одинаковый стиль рассуждений | Другой подход к логике |
