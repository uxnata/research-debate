# Evidence Verifier — Верификация цитат

Проверяет, что evidence в findings агентов соответствует исходным транскриптам. Каждая цитата сверяется с `input/transcripts.md`.

## Когда запускать

- После раунда 3, перед синтезом (автоматически в пайплайне)
- Или вручную: `/evidence-verifier`

## Вход

- `input/transcripts.md` — исходные транскрипты (источник истины)
- `output/round1-user-advocate.json` — findings Agent A, раунд 1
- `output/round1-biz-advocate.json` — findings Agent B, раунд 1
- `output/round3-user-advocate.json` (если есть) — findings Agent A, раунд 3
- `output/round3-biz-advocate.json` (если есть) — findings Agent B, раунд 3

## Протокол верификации

Для каждого finding из всех файлов:

1. Извлеки поле `evidence`
2. Найди в нём все цитаты (текст в кавычках «» или '')
3. Для каждой цитаты ищи **точное совпадение** в `input/transcripts.md`
4. Присвой статус:

| Статус | Критерий |
|--------|----------|
| **verified** | Цитата найдена дословно в транскрипте |
| **paraphrased** | Цитата перефразирована, но смысл и ключевые слова совпадают с оригиналом в транскрипте |
| **unverified** | Цитата не найдена в транскрипте, или смысл существенно искажён |

## Выход

Записывай в `output/evidence-verification.json`:

```json
{
  "agent": "evidence_verifier",
  "total_findings": 12,
  "total_quotes": 25,
  "summary": {
    "verified": 18,
    "paraphrased": 5,
    "unverified": 2
  },
  "details": [
    {
      "finding_id": "UA-1",
      "source_file": "round1-user-advocate.json",
      "status": "verified",
      "quotes": [
        {
          "finding_quote": "это в нашем бэклоге",
          "original_quote": "мне ответили что-то вроде 'это в нашем бэклоге'",
          "respondent": "Марина (R1)",
          "status": "verified"
        }
      ]
    },
    {
      "finding_id": "UA-2",
      "source_file": "round1-user-advocate.json",
      "status": "paraphrased",
      "quotes": [
        {
          "finding_quote": "команда постепенно перестала использовать продукт",
          "original_quote": "Первую неделю вся команда что-то заполняла. Через месяц заполнял только я. Через два — даже я перестал.",
          "respondent": "Дмитрий (R4)",
          "status": "paraphrased"
        }
      ]
    }
  ]
}
```

## Правила

1. **Источник истины — только `input/transcripts.md`**. Не используй context.md или другие файлы для верификации цитат.
2. **Числовые данные из `input/context.md`** — допустимы как evidence (метрики, цены). Проверяй их отдельно по context.md. Помечай как `verified` если совпадают.
3. **Не оценивай качество reasoning** — только точность цитат.
4. **Цитата считается verified**, даже если обрезана (часть предложения), при условии что обрезка не искажает смысл.
5. **Для paraphrased** — обязательно указывай оригинальную цитату из транскрипта, чтобы можно было сравнить.
6. **Finding-level status** = worst status среди его цитат (если хотя бы одна unverified — весь finding unverified).
