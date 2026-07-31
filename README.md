# Type9 API (Marlin Mint) — Examples

> **Статус:** черновик на основе PaySwiss (UAH, P2P/Acquiring), оформлен по конвенциям Type9.

## Base URL
```
https://api.marlinmint.com/v1
```

Стенд для интеграции:
```
https://api-test.marlinmint.com/v1
```

## Authentication
Все запросы требуют заголовок `Authorization` с вашим API-ключом.

```
Authorization: <your-api-key>
Content-Type: application/json
```

---

## Create Payment (Intake)

**Endpoint:** `POST /payments/checkout`

### Request (редирект на платёжную страницу)
```json
{
    "sum_total": "1000.00",
    "currency_iso": "UAH",
    "flow_type": "intake",
    "rail": "p2p",
    "order_ref": "order_12345",
    "payer_ref": "user_001",
    "payer_name": "Іван Петренко",
    "hook_url": "https://your-site.com/webhook",
    "done_url": "https://your-site.com/success"
}
```

### Request с реквизитами (no_ui)
Сервер-к-серверу, без нашей платёжной страницы: в ответе придут реквизиты
для перевода, показывать их плательщику вы будете сами.

```json
{
    "sum_total": "1000.00",
    "currency_iso": "UAH",
    "flow_type": "intake",
    "rail": "p2p",
    "order_ref": "order_1233272245",
    "payer_ref": "user_001",
    "payer_name": "Іван Петренко",
    "hook_url": "https://your-site.com/webhook",
    "no_ui": true
}
```

### Request с картой (rail: acquiring)
```json
{
    "sum_total": "1000.00",
    "currency_iso": "UAH",
    "flow_type": "intake",
    "rail": "acquiring",
    "order_ref": "order_123452292027",
    "payer_ref": "user_001",
    "hook_url": "https://your-site.com/webhook",
    "no_ui": true,
    "intake_details": {
        "issuer": "Private Bank",
        "card_input": {
            "card_number": "5246081344140382",
            "expiry_month": 2,
            "expiry_year": 29,
            "security_code": "908"
        }
    }
}
```

### Response
```json
{
    "txn_ref": "pay_in_abc123",
    "order_ref": "order_12345",
    "stage": "open",
    "sum_total": "1000.00",
    "currency_iso": "UAH",
    "network": null,
    "pay_page_url": "https://pay.marlinmint.com/pay_in_abc123",
    "started_at": "2026-07-31T10:30:00.000Z",
    "finished_at": null,
    "sum_settled": null,
    "failure_note": null,
    "valid_till": null,
    "bank_details": null
}
```

---

## Create Payment (Payout)

**Endpoint:** `POST /payments/checkout`

### Вывод на карту
```json
{
    "sum_total": "500.00",
    "currency_iso": "UAH",
    "flow_type": "payout",
    "rail": "p2p",
    "order_ref": "withdraw_67890",
    "payer_ref": "user_001",
    "hook_url": "https://your-site.com/webhook",
    "remit_details": {
        "to_name": "Іван Петренко",
        "dest_account": "5246081344140382"
    }
}
```

### Response
```json
{
    "txn_ref": "pay_out_xyz789",
    "order_ref": "withdraw_67890",
    "stage": "open",
    "sum_total": "500.00",
    "currency_iso": "UAH",
    "network": null,
    "pay_page_url": null,
    "started_at": "2026-07-31T11:00:00.000Z",
    "finished_at": null,
    "sum_settled": null,
    "failure_note": null,
    "valid_till": null,
    "bank_details": null
}
```

---

## Check Payment Status

**Endpoint:** `GET /payments/{txn_ref}/lookup`

```
GET /payments/pay_in_abc123/lookup
```

### Response — в процессе, реквизиты выданы
```json
{
    "txn_ref": "pay_in_abc123",
    "order_ref": "order_12345",
    "stage": "authorized",
    "sum_total": "1000.00",
    "currency_iso": "UAH",
    "network": null,
    "pay_page_url": "https://pay.marlinmint.com/pay_in_abc123",
    "started_at": "2026-07-31T10:30:00.000Z",
    "finished_at": null,
    "sum_settled": null,
    "failure_note": null,
    "bank_details": {
        "type": "card",
        "card_number": "4149629311879546",
        "beneficiary_name": "Приват*Іван П."
    }
}
```

### Response — успех
```json
{
    "txn_ref": "pay_in_abc123",
    "order_ref": "order_12345",
    "stage": "cleared",
    "sum_total": "1000.00",
    "currency_iso": "UAH",
    "network": null,
    "pay_page_url": "https://pay.marlinmint.com/pay_in_abc123",
    "started_at": "2026-07-31T10:30:00.000Z",
    "finished_at": "2026-07-31T10:35:00.000Z",
    "sum_settled": "970.00",
    "failure_note": null,
    "bank_details": null
}
```

### Response — отказ
```json
{
    "txn_ref": "pay_in_abc123",
    "order_ref": "order_12345",
    "stage": "unsettled",
    "sum_total": "1000.00",
    "currency_iso": "UAH",
    "network": null,
    "pay_page_url": null,
    "started_at": "2026-07-31T10:30:00.000Z",
    "finished_at": null,
    "sum_settled": null,
    "failure_note": "Payment timeout",
    "bank_details": null
}
```

---

## Check Balance

**Endpoint:** `GET /partner/holdings`

```json
{
    "amount_available": "15000.00",
    "ccy_code": "UAH"
}
```

---

## Raise Claim

**Endpoint:** `POST /payments/{txn_ref}/claim`

Оспаривание результата по платежу. Тело — `multipart/form-data`: от 1 до 3
вложений (до 50 MB каждое) и необязательный `comment`. Ответ:

```json
{
    "outcome": "accepted"
}
```

---

## Stage Values

Те же пять состояний, что в кабинете мерчанта и в CSV-выгрузке.

| Stage | Описание |
|-------|----------|
| `open` | создан, плательщик ещё не подтвердил перевод |
| `authorized` | деньги подтверждены, идёт проводка |
| `cleared` | завершён полностью |
| `part_cleared` | завершён частично, пришло меньше запрошенного |
| `unsettled` | не состоялся: отказ, отмена, истёк срок, реверс или чарджбэк |

Терминальные — `cleared`, `part_cleared`, `unsettled`. Остальные могут
измениться, опрашивать статус или ждать вебхук нужно до терминального.

---

## Error Response Format

```json
{
    "outcome": "rejected",
    "problem": {
        "reason": "invalid_input",
        "detail": "sum_total is required",
        "invalid_fields": null
    }
}
```

| Reason | Описание | HTTP |
|--------|----------|------|
| `invalid_input` | неверные параметры запроса | 400, 422 |
| `payment_rejected` | ошибка обработки платежа | 400 |
| `not_authorized` | ошибка аутентификации или домен не разрешён | 401, 403 |
| `internal_failure` | внутренняя ошибка | 500 |

`invalid_fields` заполняется только при ошибках валидации тела запроса —
список полей с описанием проблемы, иначе `null`.

---

## Webhook Notification

При смене стадии на ваш `hook_url` уходит POST.

### Проверка подписи (RSA SHA-256)

Каждый callback можно проверить с помощью RSA SHA-256 подписи для
подтверждения подлинности и целостности.

**Порядок проверки:**

1. Извлечь тело запроса как есть (точная JSON-строка, без модификаций).
2. Вычислить SHA-256 хеш тела.
3. Проверить подпись с помощью публичного ключа.

**Заголовок подписи:**
```
X-Signature
```

**Правила проверки:**

* Алгоритм: RSA + SHA256
* Кодировка: Base64
* Callback с некорректной подписью должен быть отклонён

### Пример payload
```json
{
    "txn_ref": "pay_in_abc123",
    "order_ref": "order_12345",
    "stage": "cleared",
    "sum_total": "1000.00",
    "currency_iso": "UAH",
    "network": null,
    "started_at": "2026-07-31T10:30:00.000Z",
    "finished_at": "2026-07-31T10:35:00.000Z",
    "sum_settled": "970.00",
    "failure_note": null
}
```

Отвечайте `200` — иначе доставка будет повторяться. Обработчик должен быть
идемпотентным по `txn_ref`: на один платёж может прийти несколько
уведомлений (например `authorized`, затем `cleared`).

---

## Field Reference

### Request Fields

| Field | Type | Required | Описание |
|-------|------|----------|----------|
| `sum_total` | string | да | сумма транзакции |
| `currency_iso` | string | да | код ISO 4217 (`UAH`) |
| `flow_type` | string | да | `intake` (приём) или `payout` (выплата) |
| `rail` | string | да | `p2p` (карта-в-карту / банковский перевод) или `acquiring` (эквайринг карт) |
| `network` | string | нет | платёжная сеть, если применимо |
| `order_ref` | string | да | ваш уникальный идентификатор, 5–64 символа |
| `payer_ref` | string | да | идентификатор плательщика на вашей стороне |
| `payer_name` | string | нет | имя плательщика |
| `hook_url` | string | да | URL для вебхуков |
| `done_url` | string | условно | обязателен, если `no_ui` не задан |
| `no_ui` | boolean | нет | режим без нашей платёжной страницы |
| `intake_details` | object | нет | реквизиты приёма |
| `remit_details` | object | нет | реквизиты выплаты |

`order_ref` — ключ идемпотентности. Повторный запрос с тем же `order_ref`
вернёт существующий платёж, а не создаст новый.

### intake_details (приём)

| Field | Type | Описание |
|-------|------|----------|
| `issuer` | string | банк для подбора P2P-реквизитов |
| `card_input` | object | данные карты для `rail: acquiring` |

### card_input

| Field | Type | Описание |
|-------|------|----------|
| `card_number` | string | номер карты |
| `expiry_month` | number | месяц истечения |
| `expiry_year` | number | год истечения (2 цифры) |
| `security_code` | string | CVV/CVC |

### remit_details (выплата)

| Field | Type | Описание |
|-------|------|----------|
| `dest_account` | string | номер карты, 16 цифр |
| `to_name` | string | ФИО получателя |
| `iban_code` | string | IBAN для международных переводов |
| `tax_number` | string | налоговый идентификатор |

### Response Fields

| Field | Type | Описание |
|-------|------|----------|
| `txn_ref` | string | наш идентификатор платежа |
| `order_ref` | string | ваш идентификатор из запроса |
| `stage` | string | стадия платежа, см. Stage Values |
| `sum_total` | string | запрошенная сумма |
| `sum_settled` | string \| null | фактически проведённая сумма |
| `currency_iso` | string | код ISO 4217 |
| `network` | string \| null | платёжная сеть |
| `pay_page_url` | string \| null | ссылка на платёжную страницу |
| `started_at` | string | момент создания, ISO 8601 |
| `finished_at` | string \| null | момент завершения, ISO 8601 |
| `failure_note` | string \| null | причина отказа |
| `valid_till` | string \| null | срок жизни платежа, если задан |
| `bank_details` | object \| null | реквизиты для перевода в режиме `no_ui` |
