# Vibe Broker

CLI-инструмент для яхтенных брокеров на базе Claude Code skills.

## Установка

```bash
git clone https://github.com/stephansergeev/vibe-broker.git
cd vibe-broker
cp .env.example .env
# заполни .env своими токенами
claude  # запусти Claude Code
```

## Skills

| Команда | Описание |
|---------|----------|
| `/rop` | РОП-анализатор: кто из менеджеров недорабатывает в воронке Sell My Boat |

## Конфигурация `.env`

```
KOMMO_ACCESS_TOKEN=...
KOMMO_BASE_URL=https://bygroup.kommo.com
KOMMO_SMB_PIPELINE_ID=...
```
