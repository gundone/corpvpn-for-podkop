# Документация Podkop

Локальная копия документации проекта [Podkop](https://podkop.net/docs/) -- пакета для OpenWrt для селективной маршрутизации трафика через VPN/прокси.

- GitHub: https://github.com/itdoginfo/podkop
- Документация: https://podkop.net/docs/

## Содержание

| Файл | Описание |
|------|----------|
| [install.md](install.md) | Установка, обновление, удаление, несовместимости |
| [settings.md](settings.md) | Глобальные настройки: DNS, интерфейсы, YACD, QUIC, списки |
| [sections.md](sections.md) | Секции: типы подключений (Proxy/VPN/Block/Exclusion), списки доменов и подсетей |
| [dns.md](dns.md) | Тонкости работы DNS, схема FakeIP, Domain Resolver |
| [workvpn.md](workvpn.md) | Совместимость с корпоративным VPN (Mixed Mode, VPN на роутере) |
| [dont-touch-my-dhcp.md](dont-touch-my-dhcp.md) | Ручная настройка dnsmasq, split DNS для конкретных доменов |
| [faq.md](faq.md) | Часто задаваемые вопросы |
| [troubleshooting.md](troubleshooting.md) | Диагностика и устранение неисправностей |
| [adguard.md](adguard.md) | Интеграция с AdGuard Home |

## Архитектура DNS (FakeIP)

```
Клиент :53 → dnsmasq 0.0.0.0:53 → sing-box 127.0.0.42:53 → DNS-server
```

Домены из списков получают FakeIP из диапазона `198.18.0.0/15`, трафик маркируется nftables и перенаправляется через tproxy в sing-box.
