# Диагностика и устранение блокировок на уровне ядра Linux (без UFW)

Инструкция для случаев, когда блокировки происходят БЕЗ UFW - на уровне встроенных защитных механизмов ядра Linux.

## Симптомы проблемы

- ✅ Сервер отвечает на ping
- ✅ SSH работает ЕСЛИ соединение было установлено ДО блокировки
- ❌ НОВЫЕ SSH соединения НЕ устанавливаются
- ❌ VPN subscription не обновляется
- ❌ Панель VPN недоступна через браузер
- ✅ С других IP всё работает
- ✅ Через 5-10 минут всё само восстанавливается
- ❌ UFW отключен или отсутствует

**Ключевое отличие от UFW блокировок:** Нет записей `[UFW BLOCK]` в `/var/log/ufw.log`

---

## ⚡ БЫСТРОЕ РЕШЕНИЕ

**Если не хотите читать всю инструкцию** - примените оптимизированные настройки:

```bash
# Создайте файл с оптимизациями
sudo tee /etc/sysctl.d/99-vpn-optimization.conf <<EOF
# Conntrack оптимизация
net.netfilter.nf_conntrack_max = 524288
net.netfilter.nf_conntrack_tcp_timeout_established = 7200

# TCP оптимизация
net.ipv4.tcp_max_syn_backlog = 8192
net.core.netdev_max_backlog = 4096
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_timestamps = 1

# Защита от SYN flood
net.ipv4.tcp_syn_retries = 2
net.ipv4.tcp_fin_timeout = 15

# Увеличение лимитов портов
net.ipv4.ip_local_port_range = 1024 65535
EOF

# Применить настройки
sudo sysctl -p /etc/sysctl.d/99-vpn-optimization.conf

# Установить hashsize
sudo sh -c 'echo 131072 > /sys/module/nf_conntrack/parameters/hashsize'

# Перезагрузите сервер для применения всех изменений
sudo reboot
```

---

## Диагностика и решение

### 1. Проверьте защиту от SYN flood

**На сервере выполните:**

```bash
# Проверьте логи ядра на SYN flood защиту
dmesg | grep -i "SYN flood"
journalctl -k | grep -i "cookies"
```

**Если видите:**
```
TCP: Possible SYN flooding on port 443. Sending cookies.
```

**Означает:** Ядро само обнаружило атаку и включило защиту. Это нормально, но может вызывать временные блокировки.

**Решение:**
```bash
# Проверьте текущие настройки
sysctl net.ipv4.tcp_syncookies
sysctl net.ipv4.tcp_synack_retries

# Оптимизируйте (уже включено в БЫСТРОМ РЕШЕНИИ выше)
sudo sysctl -w net.ipv4.tcp_syncookies=1
sudo sysctl -w net.ipv4.tcp_synack_retries=2
```

### 2. Проверьте переполнение conntrack таблицы

**ЭТО САМАЯ ЧАСТАЯ причина блокировок на VPN серверах!**

```bash
# Текущее количество соединений
cat /proc/sys/net/netfilter/nf_conntrack_count

# Максимальный лимит
cat /proc/sys/net/netfilter/nf_conntrack_max

# Вычислите процент заполненности
echo "scale=2; $(cat /proc/sys/net/netfilter/nf_conntrack_count) * 100 / $(cat /proc/sys/net/netfilter/nf_conntrack_max)" | bc

# Проверьте логи на ошибки
dmesg | grep -i "nf_conntrack"
journalctl -k | grep "table full"
```

**Если видите:**
```
nf_conntrack: table full, dropping packet
```

**Означает:** Таблица соединений переполнена! Новые соединения отбрасываются.

**Решение:**

```bash
# Увеличьте лимит conntrack (для 4GB RAM = 262144, для 8GB = 524288)
sudo sysctl -w net.netfilter.nf_conntrack_max=524288

# Увеличьте размер hash таблицы (должен быть = conntrack_max / 4)
sudo sh -c 'echo 131072 > /sys/module/nf_conntrack/parameters/hashsize'

# Сделайте изменения постоянными (conntrack_max)
sudo tee -a /etc/sysctl.conf <<EOF
net.netfilter.nf_conntrack_max = 524288
EOF

# Для hashsize создайте файл конфигурации модуля
sudo tee /etc/modprobe.d/nf_conntrack.conf <<EOF
options nf_conntrack hashsize=131072
EOF

# Перезагрузите сервер
sudo reboot
```

### 3. Проверьте переполнение TCP backlog очереди

```bash
# Статистика TCP
netstat -s | grep -i listen

# Ищите строки:
# XXX times the listen queue of a socket overflowed
# XXX SYNs to LISTEN sockets dropped
```

**Если значения растут** - это признак переполнения очередей.

**Решение:**

```bash
# Увеличьте размер SYN backlog
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=8192

# Увеличьте общую backlog очередь
sudo sysctl -w net.core.netdev_max_backlog=4096

# Сделайте постоянными
sudo tee -a /etc/sysctl.conf <<EOF
net.ipv4.tcp_max_syn_backlog = 8192
net.core.netdev_max_backlog = 4096
EOF

# Применить настройки
sudo sysctl -p
```

## Расчёт оптимальных значений

### nf_conntrack_max

**Формула:** `256 * RAM_GB`

| RAM сервера | Рекомендуемое значение |
|-------------|------------------------|
| 1 GB | 65536 |
| 2 GB | 131072 |
| 4 GB | 262144 |
| 8 GB | 524288 |
| 16 GB | 1048576 |

### hashsize

**Формула:** `nf_conntrack_max / 4`

Примеры:
- `nf_conntrack_max=262144` → `hashsize=65536`
- `nf_conntrack_max=524288` → `hashsize=131072`

### tcp_max_syn_backlog

**Рекомендации:**
- Малый трафик (личный VPN): 2048
- Средний трафик: 4096
- Высокий трафик (VPN/CDN): 8192-16384

## Проверка результатов

```bash
# Проверьте новые значения
sysctl net.netfilter.nf_conntrack_max
sysctl net.ipv4.tcp_max_syn_backlog
cat /sys/module/nf_conntrack/parameters/hashsize

# Мониторьте conntrack в реальном времени
watch -n 1 'echo "Conntrack: $(cat /proc/sys/net/netfilter/nf_conntrack_count) / $(cat /proc/sys/net/netfilter/nf_conntrack_max)"'

# Проверьте логи
dmesg | tail -50
journalctl -k -n 50
```

**Хорошие признаки:**
- ✅ Нет сообщений `table full, dropping packet`
- ✅ Нет сообщений `SYN flooding`
- ✅ `nf_conntrack_count` далеко от `nf_conntrack_max` (< 80%)
- ✅ Новые соединения устанавливаются стабильно

## Мониторинг

### Создайте скрипт для автоматической проверки

```bash
sudo tee /usr/local/bin/check-conntrack.sh <<'EOF'
#!/bin/bash

MAX=$(cat /proc/sys/net/netfilter/nf_conntrack_max 2>/dev/null || echo "0")
COUNT=$(cat /proc/sys/net/netfilter/nf_conntrack_count 2>/dev/null || echo "0")

if [ "$MAX" -eq 0 ]; then
  echo "⚠️  nf_conntrack module not loaded"
  exit 1
fi

PERCENT=$((COUNT * 100 / MAX))

echo "$(date '+%Y-%m-%d %H:%M:%S') - Conntrack usage: $COUNT / $MAX ($PERCENT%)"

if [ $PERCENT -gt 90 ]; then
  echo "🚨 CRITICAL: Conntrack table is ${PERCENT}% full!"
elif [ $PERCENT -gt 80 ]; then
  echo "⚠️  WARNING: Conntrack table is ${PERCENT}% full!"
fi
EOF

sudo chmod +x /usr/local/bin/check-conntrack.sh

# Запустите вручную
/usr/local/bin/check-conntrack.sh
```

### Добавьте в cron для автоматического мониторинга

```bash
# Проверять каждые 5 минут и логировать
(crontab -l 2>/dev/null; echo "*/5 * * * * /usr/local/bin/check-conntrack.sh >> /var/log/conntrack-monitor.log 2>&1") | crontab -

# Проверьте что задача добавлена
crontab -l
```

### Просмотр логов мониторинга

```bash
# Последние 20 записей
tail -20 /var/log/conntrack-monitor.log

# Следить в реальном времени
tail -f /var/log/conntrack-monitor.log
```

## Troubleshooting

### Проблема: Изменения не применяются

```bash
# Убедитесь что модуль nf_conntrack загружен
lsmod | grep nf_conntrack

# Если не загружен - загрузите
sudo modprobe nf_conntrack

# Проверьте права доступа к конфигурациям
ls -l /etc/sysctl.conf
ls -l /etc/sysctl.d/

# Примените все настройки sysctl
sudo sysctl --system
```

### Проблема: hashsize не меняется

```bash
# hashsize можно менять только при загрузке модуля
# Проверьте текущее значение
cat /sys/module/nf_conntrack/parameters/hashsize

# Создайте файл конфигурации модуля
sudo tee /etc/modprobe.d/nf_conntrack.conf <<EOF
options nf_conntrack hashsize=131072
EOF

# ОБЯЗАТЕЛЬНО перезагрузите сервер
sudo reboot
```

### Проблема: После перезагрузки настройки сбрасываются

```bash
# Проверьте что systemd-sysctl активен
sudo systemctl status systemd-sysctl

# Проверьте синтаксис всех sysctl файлов
sudo sysctl --system

# Убедитесь что файлы существуют
ls -l /etc/sysctl.d/99-vpn-optimization.conf
cat /etc/sysctl.d/99-vpn-optimization.conf
```

## Когда НЕ помогает

Если после всех настроек проблема остаётся:

### 1. Проверьте другие firewall

```bash
# Проверьте iptables
sudo iptables -L -n -v

# Проверьте nftables
sudo nft list ruleset

# Проверьте firewalld
sudo firewall-cmd --list-all
```

### 2. Проверьте fail2ban

```bash
# Проверьте ваш IP в банах
sudo fail2ban-client status sshd

# Разбаньте себя если нужно
sudo fail2ban-client set sshd unbanip YOUR_IP
```

### 3. Проверьте хостинг-провайдера

Некоторые провайдеры (Hetzner, OVH, AWS) имеют свою защиту от DDoS на уровне сети. Свяжитесь с поддержкой.

### 4. Настройте MTU

Высокий MTU вызывает фрагментацию → больше пакетов → чаще превышение лимитов.

**См. инструкцию:** [Проверка оптимального MTU для VPN](mtu-check-guide.md)

**Рекомендация:** Установите MTU = 1400 в VPN клиенте.

## Откат изменений (если нужно)

```bash
# Удалите файл оптимизаций
sudo rm /etc/sysctl.d/99-vpn-optimization.conf

# Удалите конфигурацию модуля
sudo rm /etc/modprobe.d/nf_conntrack.conf

# Удалите cron задачу мониторинга
crontab -l | grep -v check-conntrack.sh | crontab -

# Удалите скрипт
sudo rm /usr/local/bin/check-conntrack.sh

# Перезагрузите сервер для возврата к дефолтным значениям
sudo reboot
```

---

**Готово!** Теперь ваш сервер оптимизирован для работы с большим количеством VPN подключений. 🚀
