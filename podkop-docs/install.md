# Установка -- Podkop

Источник: https://podkop.net/docs/install/

## Требования
- OpenWrt 24.10
- Минимум 20MB свободного места на NAND; sing-box включен как зависимость

## Автоматическая установка и обновление
```
sh <(wget -O - https://raw.githubusercontent.com/itdoginfo/podkop/refs/heads/main/install.sh)
```

## Ручная установка из пакетов
1. Выполнить `opkg update` для зависимостей
2. Скачать `podkop_*.ipk` и `luci-app-podkop_*.ipk` из релизов
3. Установить: `opkg install <package-path>` (сначала podkop, затем luci-app-podkop)

## Установка на 23.05
Не рекомендуется. Версия 0.5.0+ требует sing-box 1.12 и jq 1.7.1. Варианты: установить релиз 0.4.11 или вручную установить sing-box/jq из пакетов 24.10.

## Установка на 25.12
Используйте автоматический скрипт или **.apk** файлы из релизов. Не протестировано; сообщайте о проблемах в чат.

## Несовместимости
1. **Getdomains script** -- несовместим; доступен скрипт удаления
2. **https-dns-proxy** -- конфликтует с конфигурацией dhcp; предоставлена команда удаления
3. **Legacy iptables packages** -- iptables-mod-extra мешает работе tproxy

## Удаление
```
opkg remove luci-i18n-podkop-ru luci-app-podkop podkop
```

## Обновление OpenWrt
Остановите podkop перед обновлением:
```
service podkop stop
```

Восстановление DNS при обновлении без остановки:
```
service podkop stop
uci -q delete dhcp.@dnsmasq[0].server
uci add_list dhcp.@dnsmasq[0].server="8.8.8.8"
uci commit dhcp
service dnsmasq restart
ntpd -q -p ptbtime1.ptb.de
service podkop start
```
