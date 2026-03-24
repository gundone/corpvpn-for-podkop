# Дружим с AdGuardHome -- Podkop

Источник: https://podkop.net/docs/adguard/

Для корректной работы Podkop необходимо обеспечить прохождение всех DNS-запросов через него.

Два способа интеграции с AdGuard Home (AGH):

1. Указать AGH как DNS-сервер в конфигурации Podkop
2. Прописать роутер как Upstream DNS-сервер в AGH, а клиентам раздать адрес AGH по DHCP

## AGH как DNS-сервер в Podkop

### AGH на роутере

Измените конфигурацию AGH в `/etc/adguardhome.yaml`:

```yaml
bind_hosts : 127.0.0.10,
port : 53
```

> **Warning:** Не используйте loopback-адреса 127.0.0.42 и 127.0.0.53 -- конфликт с другими службами

Перезапустите AGH:
```bash
service adguardhome restart
```

Настройки DNS в Podkop:
- DNS Protocol Type: UDP
- DNS Server: 127.0.0.10

### AGH на другом устройстве

Если AGH на 192.168.1.10:
- DNS Protocol Type: UDP
- DNS Server: 192.168.1.10

### Проверка
Убедитесь, что AGH принимает запросы (раздел "Журнал" в AGH).
Настройте Upstream DNS в AGH (например, `https://dns.google/dns-query`, `tls://dns.adguard-dns.com`).

## AGH как DNS-сервер у клиентов

### Принцип работы
1. Клиенты получают адрес AGH через DHCP
2. AGH фильтрует, собирает статистику и передаёт в Podkop
3. Podkop передаёт внешним DNS

### Настройка DHCP
Укажите IP устройства с AGH (например, 192.168.1.10) в поле DNS-сервера DHCP.

### Настройка AGH
1. Настройки -> Настройки DNS
2. Upstream DNS-серверы: IP роутера с Podkop (например, `192.168.1.1`)
