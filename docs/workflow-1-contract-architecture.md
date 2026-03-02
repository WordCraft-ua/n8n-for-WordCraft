# Workflow 1 — Контракт і архітектура

## 1) Контекст та ціль

Цей workflow реалізує генерацію рахунку з Google Sheets за кодом ЄДРПОУ, формує PDF через APITemplate.io, надсилає документ у Telegram та додатково відправляє текст із реквізитами партнера.

Ціль міграції: перенести логіку з n8n у керований сервіс (API/worker), з чіткими контрактами, контрольованими помилками та можливістю масштабування.

---

## 2) Бізнес-флоу (as-is)

1. Вхідне повідомлення в Telegram (`message.text`) трактується як `edrpou`.
2. Сервіс шукає запис у Google Sheet `Дані договору` за колонкою `ЄДРПОУ`.
3. Дані мапляться у DTO для шаблону PDF.
4. Викликається APITemplate.io (`resource=pdf`, `pdfTemplateId=a5077b23f48251da`).
5. За `download_url` завантажується PDF.
6. PDF надсилається як Telegram document користувачу.
7. Користувачу надсилається друге Telegram-повідомлення з деталями партнера і підказкою.

---

## 3) Контракт Workflow 1

### 3.1 Вхідний контракт

Джерело: Telegram Update (подія `message`).

Мінімально необхідні поля:

```json
{
  "message": {
    "chat": { "id": 551854787 },
    "text": "3072503284"
  }
}
```

#### Нормалізація вхідних даних

- `chatId`: `message.chat.id`
- `edrpouRaw`: `message.text`
- `edrpou`: trim + тільки цифри (рекомендовано).

### 3.2 Внутрішній контракт (Google row → Invoice DTO)

Після пошуку в таблиці формується об'єкт:

```json
{
  "invoice_no": "...",
  "date_price": "...",
  "company_bill_to": "...",
  "address": "...",
  "document_id": "...",
  "monthly_price": "...",
  "bank": "...",
  "iban": "...",
  "mfo": "...",
  "months_of_debt": "...",
  "number_of_months": "...",
  "sum": "...",
  "amount_in_writing": "...",
  "email": "...",
  "contract_number": "..."
}
```

Мапінг колонок Google Sheet:

- `Номер Рахуноку` → `invoice_no`
- `Дата` → `date_price`
- `НАЗВА` → `company_bill_to`
- `ЮРИДИЧНА АДРЕСА` → `address`
- `ЄДРПОУ` → `document_id` (string)
- `Сума в місяць` → `monthly_price`
- `НАЗВА БАНКУ/ФІЛІЇ` → `bank`
- `IBAN` → `iban`
- `мфо` → `mfo`
- `Місяці заборгованості` → `months_of_debt`
- `К-ть НЕоплачених місяців` → `number_of_months`
- `Сума заборгованості` → `sum`
- `Сума заборгованості прописом` → `amount_in_writing`
- `ЕЛЕКТРОННА СКРИНЬКА` → `email`
- `НОМЕР ДОГОВОРУ` → `contract_number`

### 3.3 Зовнішні інтеграційні контракти

#### Google Sheets (read)

- Вхід: `edrpou`
- Фільтр: `lookupColumn="ЄДРПОУ"`, `lookupValue=edrpou`
- Вихід: 0..N рядків (у цільовому сервісі потрібно обробляти обидва кейси: none / duplicate).

#### APITemplate.io (create PDF)

- Вхід: `pdfTemplateId` + `properties` із DTO.
- Вихід (очікувано): поле `download_url`.

#### HTTP download (PDF)

- Вхід: `download_url`
- Вихід: binary PDF (content-type `application/pdf`).

#### Telegram Bot API

1) `sendDocument`
- `chat_id`
- `document` (binary PDF)
- filename: `Рахунок на оплату за {months_of_debt} для {document_id}.pdf`

2) `sendMessage`
- `chat_id`
- `text` (шаблон з реквізитами)
- `parse_mode=HTML`

### 3.4 Вихідний контракт

Фактичний результат для користувача — 2 повідомлення в Telegram:

1. PDF-документ рахунку.
2. Текстовий меседж із реквізитами та інструкцією по відправці на email.

В API-представленні сервісу (рекомендовано):

```json
{
  "status": "ok",
  "chatId": 551854787,
  "edrpou": "3072503284",
  "invoice": {
    "document_id": "3072503284",
    "months_of_debt": "...",
    "sum": "..."
  },
  "delivery": {
    "documentSent": true,
    "messageSent": true
  }
}
```

### 3.5 Контракт помилок (рекомендовано для сервісу)

Стандартизований формат:

```json
{
  "status": "error",
  "code": "PARTNER_NOT_FOUND",
  "message": "За вказаним ЄДРПОУ партнера не знайдено",
  "details": {}
}
```

Коди:

- `VALIDATION_ERROR` — порожній/некоректний `edrpou`.
- `PARTNER_NOT_FOUND` — 0 рядків у Google Sheets.
- `PARTNER_DUPLICATE` — >1 рядка за `ЄДРПОУ`.
- `PDF_TEMPLATE_ERROR` — APITemplate не сформував документ.
- `PDF_DOWNLOAD_ERROR` — помилка завантаження PDF.
- `TELEGRAM_SEND_ERROR` — помилка `sendDocument` або `sendMessage`.
- `INTEGRATION_AUTH_ERROR` — проблеми доступу до Google/APITemplate/Telegram.

---

## 4) Цільова архітектура (to-be)

## 4.1 Компоненти

1. **Webhook Ingress (Telegram endpoint)**
   - Приймає update.
   - Валідує signature/token (якщо використовується).
   - Публікує job у чергу або викликає use-case синхронно.

2. **Workflow1 Orchestrator (application layer)**
   - Координує кроки бізнес-процесу.
   - Не містить SDK-специфічної логіки.

3. **Adapters / Clients (infrastructure layer)**
   - `GoogleSheetsClient`
   - `ApiTemplateClient`
   - `TelegramClient`
   - `FileDownloader`

4. **Template service**
   - Рендерить текст другого Telegram-повідомлення.
   - Підтримує версіонування/локалізацію шаблонів.

5. **Observability**
   - Structured logs (correlationId, chatId, edrpou).
   - Metrics: latency, success rate, error rate per integration.

## 4.2 Рекомендовані межі модулів

- `src/domain/invoice/` — сутності та інваріанти.
- `src/application/workflow1/` — use-case `GenerateAndSendInvoice`.
- `src/infrastructure/google/` — читання таблиці.
- `src/infrastructure/apitemplate/` — генерація PDF.
- `src/infrastructure/telegram/` — доставка в Telegram.
- `src/interfaces/http/` — webhook/controller.

## 4.3 Послідовність виконання (target)

1. Отримати Telegram update.
2. Валідовати `message.text` як `edrpou`.
3. Завантажити рівно 1 запис партнера з Google Sheets.
4. Побудувати `InvoiceDTO`.
5. Згенерувати PDF через APITemplate.
6. Завантажити PDF-байти.
7. Відправити `sendDocument`.
8. Відправити `sendMessage`.
9. Записати structured event `workflow1.completed`.

## 4.4 Надійність та ретраї

- Retry із exponential backoff для:
  - APITemplate create PDF
  - PDF download
  - Telegram send
- Без retry для `VALIDATION_ERROR`/`PARTNER_NOT_FOUND`.
- Idempotency key: `telegramUpdateId` (щоб не дублювати відправки при повторній доставці webhook).

## 4.5 Конфігурація

Обов'язкові ENV:

- `TELEGRAM_BOT_TOKEN`
- `GOOGLE_SHEETS_ID`
- `GOOGLE_SHEET_GID_OR_NAME`
- `APITEMPLATE_API_KEY`
- `APITEMPLATE_TEMPLATE_ID`
- `APP_TIMEZONE=Europe/Kyiv`

## 4.6 Безпека

- Секрети лише через env/secret manager.
- Логи без PII-секретів (маскувати токени, ключі, частково email).
- Валідація довжини/формату вхідного `message.text`.

---

## 5) План міграції Workflow 1

1. Описати DTO + error model (цей документ).
2. Реалізувати adapters для трьох інтеграцій.
3. Реалізувати use-case `GenerateAndSendInvoice`.
4. Додати webhook endpoint під Telegram.
5. Покрити unit-тестами:
   - мапінг row → DTO
   - branch помилок
   - формування текстового повідомлення
6. Додати integration/e2e тести на моках інтеграцій.
7. Запустити shadow mode (n8n + сервіс паралельно, без дублю відправки).
8. Cutover: перемкнути Telegram webhook на новий сервіс.

---

## 6) Відкриті питання (що ще треба підтвердити)

1. Поведінка при `PARTNER_NOT_FOUND` (точний текст відповіді користувачу).
2. Поведінка при дублікаті `ЄДРПОУ`.
3. Чи потрібно логувати/зберігати PDF у сховище (S3/GCS) для аудиту.
4. Чи потрібен окремий канал alerting (Telegram admin/Slack) на помилки інтеграцій.
5. Чи фіксуємо шаблон повідомлення як конфігурований ресурс (а не hardcoded).
