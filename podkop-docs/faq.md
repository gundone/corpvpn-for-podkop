# Часто задаваемые вопросы -- Podkop

Источник: https://podkop.net/docs/faq/

## Установка и обновление

### Как установить Podkop?
```
sh <(wget -O - https://raw.githubusercontent.com/itdoginfo/podkop/refs/heads/main/install.sh)
```

### APK поддерживается?
Да, используйте ту же команду автоматической установки.

### Как правильно обновить sing-box?

**Вариант 1: Обновление через скрипт Podkop**
Скрипт автоматической установки обновляет sing-box если текущая версия ниже минимально необходимой.

**Вариант 2: Обновление вручную**
```
service podkop stop
opkg update
opkg upgrade sing-box
service podkop start
```

### Как устранить ошибку нехватки места?

**Решение 1: --force-space**
```
opkg upgrade sing-box --force-space
```

**Решение 2: sing-box-tiny**
```
service podkop stop
opkg update
opkg remove sing-box --force-depends
opkg install sing-box-tiny
service podkop start
```

**Решение 3: Собрать свою прошивку**
Используйте официальный селектор прошивок OpenWrt. Включите в пакеты: `libc`, `ca-bundle`, `kmod-inet-diag`, `kmod-tun`, `sing-box`.

### Будет ли работать Podkop на кастомных прошивках?

Протестирован только на OpenWrt 24.10. Известные проблемы:
- **FriendlyWrt:** `br-netfilter` использует `iptables` -- исправлено в Podkop v0.4.6+
- **GL.Inet (v4.8.*-op24):** Конфликт с `vpn-client` -- отключите правила
- **Koshev build:** Нет `nftables` -- не будет работать
- **acdev / remittor:** Нет `sing-box`, есть конфликтующие пакеты
- **ImmortalWrt:** Альтернативные репозитории могут быть недоступны

### Как устранить ошибку Operation not permitted?

1. Проверьте время на роутере: `date`, при необходимости `ntpd -p 194.190.168.1`
2. Проверьте DNS: `nslookup downloads.openwrt.org`
3. Отключите IPv6 при необходимости

### Как откатиться на старую версию?
Скачайте *.ipk файлы нужного релиза и установите вручную.

### После остановки Podkop не работает интернет

1. Проверьте, включена ли опция **Use DNS servers advertised by peer** в WAN
2. Или укажите DNS-сервер в LuCI -> DHCP and DNS -> Forwards (например, 8.8.8.8)

### Можно ли загрузить подписку?
Встроенной поддержки пока нет. Извлеките отдельные ссылки и вставьте в секции или URLTest.

---

## Настройка маршрутизации и списков

### Как направить весь трафик устройства через VPN/прокси?

Для одного устройства:
- Назначьте статический IP
- Добавьте IP в список полного перенаправления в Podkop

Для всех устройств (не полностью протестировано):
```
uci set podkop.main.all_traffic_from_ip_enabled='1'
uci add_list podkop.main.all_traffic_ip='192.168.1.0/24'
uci commit podkop
service podkop restart
```

### Как исключить домен из списка?
Создайте секцию с типом **Exclusion** и добавьте домены.

### Как исключить устройство из туннеля?
Используйте опцию **Исключенные из маршрутизации IP-адреса**.

### Как заставить работать звонки в Telegram, WhatsApp, Discord?
Добавьте списки: Telegram, Meta*, Discord. Отключите p2p-звонки.

### Как настроить Gemini?
Включите список `Google AI` и проверьте определение страны VPS.

### Как настроить ChatGPT?
Сервисы OpenAI в списках `Russia inside` и `Geoblock`.

### Как использовать разные DNS для разных доменов?
- **Домены из списков (прокси):** DNS резолвится на сервере, нельзя выбрать через Podkop
- **Домены из списков (VPN):** Используйте Domain Resolver в настройках секции
- **Домены не из списков:** Используется глобальный DNS-сервер
- **Гибкая настройка:** Используйте Don't Touch My DHCP + dnsmasq server directives

### Как настроить Podkop с корпоративным VPN?
Используйте Mixed mode + прокси в браузере, или корп. VPN на роутере. Подробнее: [workvpn.md](workvpn.md)

### Как отправить торренты в обход Podkop?
Создайте секцию VPN с WAN-интерфейсом, включите mixed proxy, настройте торрент-клиент на этот прокси.

---

## Гостевая Wi-Fi сеть

### Гостевая НЕ использует Podkop
Создайте отдельный сетевой интерфейс и не включайте его в Source Network interface.

### Гостевая использует Podkop
1. Создайте гостевую сеть
2. Настройте firewall: Input и Infra Zone Forward = accept
3. Создайте Traffic Rule
4. В Podkop установите Source Network Interface на br-guest
