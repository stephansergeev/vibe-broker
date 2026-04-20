# /rop — РОП-анализатор воронки Sell My Boat

Анализируй воронку "Sell My Boat" (pipeline_id=7212056) в Kommo CRM.

**Цели:**
1. Все лиды сконтактированы (нет зависших в автоматических этапах)
2. Все лиды на правильных этапах (ненужные закрыты)
3. Брокеры в курсе своих лидов — сформируй им Telegram-сводки

## Шаг 1: Загрузи переменные

```bash
source "$(git rev-parse --show-toplevel)/.env"
```

## Шаг 2: Получи все открытые лиды SMB через Python

```python
import urllib.request, json, time

TOKEN = "<KOMMO_ACCESS_TOKEN из .env>"
BASE = "https://bygroup.kommo.com"
PIPELINE = 7212056

def get(url):
    req = urllib.request.Request(url, headers={"Authorization": f"Bearer {TOKEN}"})
    with urllib.request.urlopen(req, timeout=15) as r:
        return json.loads(r.read())

all_leads = []
page = 1
while True:
    data = get(f"{BASE}/api/v4/leads?filter[pipeline_id]={PIPELINE}&filter[closed]=0&limit=250&page={page}")
    leads = data.get("_embedded", {}).get("leads", [])
    all_leads.extend(leads)
    if len(leads) < 250:
        break
    page += 1

# Фильтруем только открытые (не закрытые)
open_leads = [l for l in all_leads if l.get("status_id") not in (142, 143)]
```

## Шаг 3: Получи менеджеров

```python
data = get(f"{BASE}/api/v4/users?limit=250")
users = {u["id"]: u["name"] for u in data.get("_embedded", {}).get("users", [])}
```

## Шаг 4: Классифицируй лиды

Этапы SMB воронки:

**Анализируемые (ранние — где клиенты зависают до листинга):**
- `59701376` Incoming leads — новый, нужен первый контакт
- `98245456` Lead Distribution — не назначен брокеру
- `59701380` Request Received — ждёт ответа брокера
- `74968804` Welcome Request Sent (Auto) — автоответ ушёл, нужен живой контакт
- `74968808` 1st Response Received (Auto) — клиент ответил боту, нужен брокер
- `94801580` Market Research Sent — отправили исследование, ждём решения
- `94801584` Handed To Work (Auto) — передан брокеру автоматически, нужно подтверждение
- `102187636` Decision On Hold — думает, нужен follow-up
- `59701384` Offer to List Sent — оффер отправлен, ждём подписания
- `59701388` Listing Pending — ждём документы/фото для листинга
- `69371836` Under Offer — под офером, нужен контроль

**Исключаемые из анализа (Listed + Not Listed — живут своей жизнью):**
- `64218720` Just Listed <3M
- `64218820` Listed >3M
- `64218824` Listed >6M
- `69371832` Listed >1Y
- `95769480` Not Listed, Keep Contact

**Несконтактированные** (не дошёл до живого брокера):
`59701376, 98245456, 59701380, 74968804, 74968808, 94801584`

**Проблемные по времени** (для анализируемых этапов):
- inactive_days = (now - updated_at) / 86400
- >7д — требуют внимания
- >14д — критично
- >30д — фактически заброшены

## Шаг 5: Получи последнее сообщение по каждому лиду

```
GET /api/v4/leads/{id}/notes?limit=5&order[id]=desc
```

Из ответа — первая не-attachment заметка:
- `params.text` или `params.subject` — текст (обрезать до 80 символов)
- `created_by = 0` → "Система/клиент", иначе → имя брокера из users

## Шаг 6: Определи следующее действие по стадии

| Стадия | Следующее действие |
|--------|-------------------|
| Incoming leads | Первый контакт — позвонить / написать |
| Lead Distribution | Назначить ответственного брокера |
| Request Received | Ответить на запрос клиента |
| Welcome Sent (Auto) | Живой звонок / сообщение (автоответ уже ушёл) |
| 1st Response (Auto) | Взять в работу — клиент ответил боту |
| Market Research Sent | Follow-up — получить решение по market research |
| Handed To Work (Auto) | Подтвердить принятие лида, начать работу |
| Decision On Hold | Follow-up — уточнить решение, дедлайн |
| Offer to List Sent | Получить подписание оффера / уточнить статус |
| Listing Pending | Запросить документы и фото для листинга |
| Under Offer | Контроль оферного процесса |

## Шаг 7: Выведи таблицу по каждому менеджеру

Для каждого менеджера — отдельный блок, строки отсортированы по убыванию дней без активности:

```
=== [Менеджер] (N лидов) ===

[флаг] [Название лида]
       Стадия: X | Дд без контакта
       Последнее: [От кого]: [текст сообщения]
       Действие: [следующий шаг]
       [ссылка]
```

**Флаги:** 🔴 несконтактирован · ⚠️ заброшен >30д · 🟡 неактивен >14д · ⏰ >7д · ✅ норма

### Рекомендации РОПу

- Кто самый проблемный и почему
- Какие лиды закрыть (автоматические этапы без живого контакта + давно без активности)
- Кому нужна помощь или перераспределение

## Важно

- Используй `filter[pipeline_id]={PIPELINE}` (с квадратными скобками) — иначе API вернёт все лиды
- Ссылка на лид: `https://bygroup.kommo.com/leads/detail/{id}`
- `updated_at` — unix timestamp
- Пагинация: листай страницы пока `len(leads) == 250`
