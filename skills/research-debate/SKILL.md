# Research Debate — мультиагентные дебаты для продуктовых исследований

## Что это

Набор скиллов для автономного анализа качественных исследовательских данных (транскрипты интервью, заметки, метрики оттока) через adversarial debate трёх аналитиков с разными линзами.

Паттерн: **executor никогда не ревьюит свою работу** — критический анализ всегда приходит от другой модели или другого промпта.

## Зачем

Один LLM-чат, в который залиты все транскрипты, даёт плоский, предсказуемый отчёт. Мультиагентные дебаты создают **намеренный конфликт интерпретаций**, который выявляет слепые зоны, ложную каузальность и confirmation bias.

## Архитектура

```
               ┌─────────────────┐
               │  Данные (input/) │
               └────────┬────────┘
                        │
            ┌───────────┴───────────┐
            ▼                       ▼
   ┌─────────────────┐   ┌─────────────────┐
   │ Адвокат          │   │ Адвокат          │
   │ пользователя     │   │ бизнеса          │
   │ (user-advocate)   │   │ (biz-advocate)   │
   └────────┬─────────┘   └────────┬─────────┘
            │                       │
            └───────────┬───────────┘
                        ▼
               ┌─────────────────┐
               │ Критик           │
               │ (critic)         │
               └────────┬────────┘
                        ▼
               ┌─────────────────┐
               │ Человек решает   │
               │ (checkpoint)     │
               └────────┬────────┘
                        ▼
               ┌─────────────────┐
               │ Раунд ответов    │
               │ (A и B отвечают) │
               └────────┬────────┘
                        ▼
               ┌─────────────────┐
               │ Синтез           │
               │ (synthesis)      │
               └─────────────────┘
```

## Как запустить

### Подготовка

1. Создайте рабочую директорию:
```
research-debate/
├── input/
│   ├── transcripts.md      # транскрипты интервью
│   └── context.md          # контекст проекта, метрики
├── output/                  # создаётся автоматически
└── skills/                  # скопируйте скиллы сюда
    ├── research-debate/SKILL.md        (этот файл)
    ├── user-advocate/SKILL.md
    ├── biz-advocate/SKILL.md
    ├── critic/SKILL.md
    └── synthesis/SKILL.md
```

2. Положите данные в `input/`:
   - `transcripts.md` — транскрипты интервью, разделённые `---`
   - `context.md` — цель исследования, метрики, бизнес-контекст

### Запуск полного пайплайна

```
/research-debate
```

Или пошагово:
```
/user-advocate
/biz-advocate
/critic
/synthesis
```

### Cross-model режим (рекомендуется)

Для настоящего adversarial review используйте разные модели:
- **Executor** (Адвокат пользователя + Адвокат бизнеса): Claude Code (текущая сессия)
- **Reviewer** (Критик): внешняя модель через OpenRouter API (curl)

Требования:
- Переменная окружения `OPENROUTER_API_KEY`

Это устраняет проблему self-review blind spots.

## Протокол полного пайплайна

### Раунд 1: Параллельный анализ

1. Прочитай `input/transcripts.md` и `input/context.md`
2. Выполни `/user-advocate` — запиши результат в `output/round1-user-advocate.json`
3. Выполни `/biz-advocate` — запиши результат в `output/round1-biz-advocate.json`

> Порядок не важен. Если есть cross-model MCP, можно запустить одного агента через MCP параллельно.

### Раунд 2: Критика

4. Прочитай оба файла из раунда 1
5. Проверь `CROSS_MODEL_CRITIC` в CLAUDE.md:
   - Если `true` → выполни `/cross-model-critic` (внешняя модель через OpenRouter curl)
   - Если `false` → выполни `/critic` (self-review, менее надёжно)
6. Запиши результат в `output/round2-critic.json`

> **Рекомендация:** всегда используй cross-model. Self-review создаёт слепые зоны (Claude критикует свои же паттерны мышления).

### Чекпоинт: Человек или Авто-фокус

#### Если AUTO_PROCEED: false (по умолчанию)

6. Выведи сводку в консоль:
   - Сколько challenges от Критика
   - Какие findings устояли (strongest_findings)
   - Какие blind spots обнаружены
7. Спроси человека: "Какое направление для следующего раунда?" (или "Завершить?")
8. Запиши ответ в `output/human-direction.md`

#### Если AUTO_PROCEED: true

6. Критик генерирует поле `"recommended_focus"` в своём JSON-выходе — это становится направлением для раунда 3
7. Запиши `recommended_focus` в `output/human-direction.md`
8. Цикл продолжается автоматически, пока:
   - Есть challenges с severity `critical` И текущий раунд < MAX_ROUNDS
9. Стоп-условия (переход к синтезу):
   - Все critical challenges получили ответ `defend` или `refine` — агенты отстояли позиции
   - Достигнут MAX_ROUNDS
   - Нет critical challenges
10. Если в ответах на критику появились НОВЫЕ critical challenges (от следующего раунда критики) — ещё раунд

### Раунд 3+: Ответы

9. Прочитай критику и направление (от человека или auto-focus)
10. Выполни `/user-advocate` в режиме ответа — `output/round3-user-advocate.json`
11. Выполни `/biz-advocate` в режиме ответа — `output/round3-biz-advocate.json`

> При AUTO_PROCEED: true и наличии critical challenges → повторная критика → повторные ответы (до MAX_ROUNDS).

### Evidence Verification

12. Выполни `/evidence-verifier` — проверь все цитаты в findings
13. Запиши результат в `output/evidence-verification.json`

### Синтез

14. Выполни `/synthesis` — собери все раунды в финальный отчёт
15. Запиши `output/RESEARCH_REPORT.md`

## Конфигурация

В `CLAUDE.md` или в начале сессии:

```markdown
## Research Debate Config

CROSS_MODEL_CRITIC: true          # Критик через внешнюю модель
CRITIC_MODEL: openai/gpt-4o      # модель OpenRouter для критики
AUTO_PROCEED: false               # true = без чекпоинтов
MAX_ROUNDS: 3                     # максимум раундов дебатов
LANGUAGE: ru                      # язык вывода
```

## Выходные файлы

```
output/
├── round1-user-advocate.json    # анализ Agent A
├── round1-biz-advocate.json     # анализ Agent B
├── round2-critic.json           # критика Agent C
├── human-direction.md           # направление от человека (или auto-focus)
├── round3-user-advocate.json    # ответ A на критику
├── round3-biz-advocate.json     # ответ B на критику
├── evidence-verification.json   # верификация цитат
├── RESEARCH_REPORT.md           # финальный отчёт
├── DIVERGENCE_MAP.md            # нерешённые расхождения
└── ASSUMPTIONS_LOG.md           # все допущения агентов
```
