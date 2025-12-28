# Автоматический мониторинг UFW через Telegram

Настройте автоматическую отправку статистики UFW в Telegram для контроля блокировок и безопасности сервера.

## Что показывает скрипт

- 📊 Количество блокировок за период
- 🌐 Топ-10 заблокированных IP адресов
- 🎯 Топ-3 заблокированных порта
- 🚫 Статус fail2ban
- 🛡️ Общий статус UFW

## Пример отчёта в Telegram

```
✅ [server-vpn] UFW Report
📊 Блокировок в текущем ufw.log (с 2025-12-20, за 8 дн.): 1247
🌐 Топ IP: 45.142.120.10 (324) 185.220.101.5 (156) 92.63.197.15 (89)
🎯 Топ порты: 22 (543) 443 (298) 3389 (156)
🚫 fail2ban (текущих банов): 3
🛡️ Status: active
```

**Расшифровка:**
- **Блокировок** - сколько раз UFW заблокировал пакеты с момента последней ротации лога
- **Топ IP** - самые активные атакующие (число в скобках = количество попыток)
- **Топ порты** - какие порты чаще всего сканируют
- **fail2ban** - сколько IP сейчас забанено fail2ban
- **Status** - статус UFW (active/inactive)

## Установка

### 1. Создайте Telegram бота

**Шаг 1:** Найдите [@BotFather](https://t.me/BotFather) в Telegram

**Шаг 2:** Отправьте команду `/newbot`

**Шаг 3:** Следуйте инструкциям и получите **TOKEN**

Пример TOKEN: `1234567890:ABCdefGHIjklMNOpqrsTUVwxyz`

**Шаг 4:** Получите свой **CHAT_ID**:

1. Напишите вашему боту любое сообщение
2. Откройте в браузере (замените YOUR_TOKEN):
   ```
   https://api.telegram.org/botYOUR_TOKEN/getUpdates
   ```
3. Найдите значение `"chat":{"id":123456789}` - это ваш CHAT_ID

### 2. Создайте скрипт на сервере

```bash
sudo nano /usr/local/bin/ufw-telegram-report.sh
```

### 3. Вставьте код скрипта

**⚠️ Замените:**
- `YOUR_TOKEN` - на токен вашего бота
- `YOUR_CHAT_ID` - на ваш chat ID
- `YOUR_HOSTNAME` - на имя вашего сервера (например: server-vpn, vpn-germany)

```bash
#!/bin/bash

TELEGRAM_TOKEN="YOUR_TOKEN"
CHAT_ID="YOUR_CHAT_ID"
HOSTNAME="YOUR_HOSTNAME"

# Пути к логам и статусу logrotate
UFW_LOG="/var/log/ufw.log"
LOGROTATE_STATUS="/var/lib/logrotate/status"

# Дата последней ротации ufw.log (формат YYYY-MM-DD)
LAST_ROTATE_DATE=$(sudo grep "\"${UFW_LOG}\"" "${LOGROTATE_STATUS}" 2>/dev/null | awk '{print $2}' | cut -d'-' -f1-3)
if [ -z "$LAST_ROTATE_DATE" ]; then
  LAST_ROTATE_DATE=$(date -d '7 days ago' '+%Y-%m-%d')
fi
DAYS_SINCE_ROTATE=$(( ( $(date +%s) - $(date -d "${LAST_ROTATE_DATE}" +%s) ) / 86400 ))

# Статистика по UFW за весь текущий ufw.log
TOTAL_BLOCK=$(sudo grep 'UFW BLOCK' "${UFW_LOG}" 2>/dev/null | wc -l)

# Топ IP (SRC=...) по всему логу
TOP_IP=$(sudo grep 'UFW BLOCK' "${UFW_LOG}" 2>/dev/null \
  | sed -n 's/.*SRC=\([^ ]*\).*/\1/p' \
  | sort | uniq -c | sort -nr | head -10 \
  | awk '{print $2" ("$1")"}' | tr '\n' ' ')

# Топ порты (DPT=...) по всему логу
TOP_PORT=$(sudo grep 'UFW BLOCK' "${UFW_LOG}" 2>/dev/null \
  | sed -n 's/.*DPT=\([0-9]*\).*/\1/p' \
  | sort | uniq -c | sort -nr | head -3 \
  | awk '{print $2" ("$1")"}' | tr '\n' ' ')

# UFW кратко
UFW_LINE=$(sudo ufw status | head -1)

# fail2ban статистика
FAIL2BAN_STATUS=$(sudo fail2ban-client status sshd 2>/dev/null | grep "Currently banned" | awk '{print $NF}')

# Сообщение
MSG="✅ [${HOSTNAME}] UFW Report
📊 Блокировок в текущем ufw.log (с ${LAST_ROTATE_DATE}, за ${DAYS_SINCE_ROTATE} дн.): ${TOTAL_BLOCK}
🌐 Топ IP: ${TOP_IP}
🎯 Топ порты: ${TOP_PORT}
🚫 fail2ban (текущих банов): ${FAIL2BAN_STATUS}
🛡️ ${UFW_LINE}
"

curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage" \
  -d chat_id="${CHAT_ID}" \
  -d text="${MSG}" > /dev/null
```

**Сохраните файл:** Ctrl+X, затем Y, затем Enter

### 4. Выдайте права на выполнение

```bash
sudo chmod +x /usr/local/bin/ufw-telegram-report.sh
```

### 5. Проверьте работу скрипта

```bash
sudo /usr/local/bin/ufw-telegram-report.sh
```

**Должно прийти сообщение в Telegram с отчётом!** ✅

### 6. Добавьте в cron для автоматической отправки

```bash
# Откройте crontab
sudo crontab -e
```

**Добавьте одну из строк в конец файла:**

```bash
# Вариант 1: Каждый день в 9:00 утра
0 9 * * * /usr/local/bin/ufw-telegram-report.sh

# Вариант 2: Каждые 12 часов
0 */12 * * * /usr/local/bin/ufw-telegram-report.sh

# Вариант 3: Каждые 6 часов
0 */6 * * * /usr/local/bin/ufw-telegram-report.sh

# Вариант 4: Каждый понедельник в 10:00
0 10 * * 1 /usr/local/bin/ufw-telegram-report.sh
```

**Сохраните:** Ctrl+X, затем Y, затем Enter

### 7. Проверьте что задача добавлена

```bash
sudo crontab -l
```

## Настройка расписания cron

| Расписание | Cron выражение | Описание |
|------------|----------------|----------|
| Каждый день в 9:00 | `0 9 * * *` | Утренний отчёт |
| Каждые 12 часов | `0 */12 * * *` | Два раза в день (00:00 и 12:00) |
| Каждые 6 часов | `0 */6 * * *` | Четыре раза в день |
| Каждые 3 часа | `0 */3 * * *` | Восемь раз в день |
| Каждый понедельник в 10:00 | `0 10 * * 1` | Еженедельный отчёт |
| Каждое 1-е число месяца в 9:00 | `0 9 1 * *` | Ежемесячный отчёт |

## Troubleshooting

### Отчёт не приходит

**Проверьте права на файл:**
```bash
ls -l /usr/local/bin/ufw-telegram-report.sh
```

Должно быть: `-rwxr-xr-x` (исполняемый файл)

**Если права неправильные:**
```bash
sudo chmod +x /usr/local/bin/ufw-telegram-report.sh
```

### Ошибки в скрипте

**Запустите с отладкой:**
```bash
sudo bash -x /usr/local/bin/ufw-telegram-report.sh
```

Покажет подробный вывод каждой команды и место ошибки.

### Проверьте логи cron

```bash
sudo grep CRON /var/log/syslog | tail -20
```

Покажет запускался ли скрипт и были ли ошибки.

### Ошибка "curl: command not found"

**Установите curl:**
```bash
sudo apt update
sudo apt install curl -y
```

### Бот не отвечает

**Проверьте TOKEN и CHAT_ID:**
```bash
# Протестируйте отправку вручную (замените YOUR_TOKEN и YOUR_CHAT_ID)
curl -X POST "https://api.telegram.org/botYOUR_TOKEN/sendMessage" \
  -d chat_id="YOUR_CHAT_ID" \
  -d text="Test message"
```

Если получили ответ `{"ok":true,...}` - TOKEN и CHAT_ID правильные.

## Модификация скрипта

### Изменить количество IP в топе

Замените `head -10` на нужное число:
```bash
# Топ-5 вместо Топ-10
| sort | uniq -c | sort -nr | head -5 \
```

### Изменить количество портов в топе

Замените `head -3` на нужное число:
```bash
# Топ-5 вместо Топ-3
| sort | uniq -c | sort -nr | head -5 \
```

### Добавить уведомление при большом количестве блокировок

Добавьте после строки `TOTAL_BLOCK=...`:
```bash
# Предупреждение если больше 1000 блокировок
if [ $TOTAL_BLOCK -gt 1000 ]; then
  MSG="⚠️ ВНИМАНИЕ: Высокая активность! ${MSG}"
fi
```

## Удаление

**Если нужно удалить мониторинг:**

```bash
# Удалите задачу из cron
sudo crontab -e
# Удалите строку с /usr/local/bin/ufw-telegram-report.sh

# Удалите скрипт
sudo rm /usr/local/bin/ufw-telegram-report.sh
```

---

**Готово!** Теперь вы будете получать регулярные отчёты о безопасности сервера в Telegram. 📊✅
