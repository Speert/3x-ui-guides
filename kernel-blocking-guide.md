# Блокировки на уровне ядра Linux (без UFW)

Для пользователей, у которых блокировки происходят БЕЗ UFW.

## Симптомы

- ✅ Ping работает
- ✅ SSH работает (если было подключено ДО блокировки)
- ❌ НОВЫЕ SSH соединения не работают
- ❌ VPN subscription не обновляется
- ✅ С других IP всё работает
- ✅ Через 5-10 минут восстанавливается

## Причины

1. **Переполнение conntrack** (самая частая!)
2. SYN flood защита
3. TCP backlog переполнен

## Быстрая диагностика

```bash
# Проверьте conntrack
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# Процент заполнения
echo "scale=2; $(cat /proc/sys/net/netfilter/nf_conntrack_count) * 100 / $(cat /proc/sys/net/netfilter/nf_conntrack_max)" | bc

# Проверьте логи
dmesg | grep "table full"
dmesg | grep "SYN flood"
```

## БЫСТРОЕ РЕШЕНИЕ

```bash
# 1. Увеличьте conntrack
sudo sysctl -w net.netfilter.nf_conntrack_max=524288
sudo sh -c 'echo 131072 > /sys/module/nf_conntrack/parameters/hashsize'

# 2. Увеличьте TCP backlog
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=8192
sudo sysctl -w net.core.netdev_max_backlog=4096

# 3. Проверьте
echo "Conntrack: $(cat /proc/sys/net/netfilter/nf_conntrack_count) / $(cat /proc/sys/net/netfilter/nf_conntrack_max)"
```

## Постоянная настройка

```bash
sudo tee /etc/sysctl.d/99-vpn-optimization.conf <<EOF
net.netfilter.nf_conntrack_max = 524288
net.netfilter.nf_conntrack_tcp_timeout_established = 7200
net.ipv4.tcp_max_syn_backlog = 8192
net.core.netdev_max_backlog = 4096
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_timestamps = 1
net.ipv4.ip_local_port_range = 1024 65535
EOF

sudo sysctl -p /etc/sysctl.d/99-vpn-optimization.conf
sudo sh -c 'echo 131072 > /sys/module/nf_conntrack/parameters/hashsize'
sudo reboot
```

## Расчёт значений

| RAM | conntrack_max | hashsize |
|-----|---------------|----------|
| 1 GB | 131072 | 32768 |
| 2 GB | 262144 | 65536 |
| 4 GB | 524288 | 131072 |
| 8 GB | 1048576 | 262144 |

## Мониторинг

```bash
sudo tee /usr/local/bin/check-conntrack.sh <<'EOF'
#!/bin/bash
MAX=$(cat /proc/sys/net/netfilter/nf_conntrack_max)
COUNT=$(cat /proc/sys/net/netfilter/nf_conntrack_count)
PERCENT=$((COUNT * 100 / MAX))
echo "$(date) - Conntrack: $COUNT/$MAX ($PERCENT%)"
[ $PERCENT -gt 90 ] && echo "🔴 CRITICAL!" || echo "✅ OK"
EOF

sudo chmod +x /usr/local/bin/check-conntrack.sh
(crontab -l; echo "*/5 * * * * /usr/local/bin/check-conntrack.sh >> /var/log/conntrack.log") | crontab -
```

## Troubleshooting

### hashsize не меняется

```bash
sudo tee /etc/modprobe.d/nf_conntrack.conf <<EOF
options nf_conntrack hashsize=131072
EOF
sudo reboot
```

### Модуль не загружен

```bash
sudo modprobe nf_conntrack
echo "nf_conntrack" | sudo tee -a /etc/modules
```

## Связь с другими проблемами

- **Если есть UFW:** используйте обе инструкции [ufw-blocking-troubleshooting.md](ufw-blocking-troubleshooting.md)
- **Высокий MTU:** MTU=9000 → больше пакетов → быстрее переполняется conntrack [mtu-check-guide.md](mtu-check-guide.md)
- **Мониторинг:** [ufw-monitoring-telegram.md](ufw-monitoring-telegram.md)

---

**Комплексное решение:**
1. ✅ Увеличьте conntrack (эта инструкция)
2. ✅ Whitelist в UFW (если есть)
3. ✅ MTU=1400
4. ✅ Telegram мониторинг
