# OpenConnect + Podkop на OpenWrt: пошаговая инструкция

Router: AX6S, OpenWrt 24.10.1
Корп. VPN: Cisco AnyConnect (OpenConnect-совместимый)
Аутентификация: пароль + 2FA (push на телефон)

---

## Шаг 0. Важные ограничения

**2FA (push на телефон):**
- OpenConnect на роутере работает в неинтерактивном режиме (`--non-inter`)
- Пароль и второй фактор передаются через stdin
- Если сервер принимает слово `push` как второй пароль — всё работает автоматически
- Если сервер использует SAML/ADFS + Duo HTML-формы — **не будет работать** без обходных путей
- При неудачной 2FA OpenConnect может агрессивно переподключаться и **заблокировать аккаунт**

**DNS конфликт:**
- Опция `peerdns '0'` **не работает** для OpenConnect! Скрипт `vpnc-script` обходит её и пишет DNS конфиг напрямую в `/tmp/dnsmasq.d/`
- Это сломает Podkop, если не исправить вручную (см. Шаг 3)

---

## Шаг 1. Установка пакетов

SSH на роутер:

```bash
opkg update
opkg install openconnect luci-proto-openconnect
```

Перезапустить rpcd для появления в LuCI:

```bash
service rpcd restart
```

Проверить, что установилось:

```bash
opkg list-installed | grep openconnect
```

---

## Шаг 2. Тестовое подключение из CLI

**Это самый важный шаг.** Нужно понять, как именно ваш сервер обрабатывает 2FA.

SSH на роутер и запустить:

```bash
openconnect --user=ВАШ_ЛОГИН https://vpn.your-corp.com
```

Наблюдайте за процессом аутентификации:

### Вариант A: Сервер спрашивает пароль, потом второй фактор

```
Password: ****
Second password:
```

В этом случае на второй запрос вводите `push` (или `phone`, `sms`, TOTP-код — зависит от настроек сервера). Если после ввода `push` на телефон приходит подтверждение и VPN подключается — **отлично, `password2 'push'` будет работать**.

### Вариант B: Сервер спрашивает только пароль и автоматически шлёт push

```
Password: ****
Waiting for token...
```

Подтвердите на телефоне. Если подключается — **password2 не нужен**.

### Вариант C: Ошибка с Duo формой

```
Unknown form (name '(null)', id 'duo_form')
```

**Плохие новости** — стандартный OpenConnect не может обработать такую 2FA. Варианты:
- Попробовать `--useragent='AnyConnect'` (иногда сервер отдаёт другую форму)
- Использовать cookie-based подход (см. раздел "Обходной путь для сложной 2FA" в конце)
- Использовать Mixed Mode в Podkop вместо OpenConnect на роутере

### После успешного подключения

Запишите:
1. Хэш сертификата сервера (будет в выводе, формат `sha256:...`)
2. Имя группы (если сервер спрашивает GROUP)
3. Какие DNS-серверы и маршруты пришли от VPN

**Отключитесь** нажатием `Ctrl+C`.

Проверьте, какие маршруты пришли от VPN (до отключения):

```bash
ip route show | grep vpn
# или
ip route show table all | grep tun
```

---

## Шаг 3. Защита DNS от перезаписи (КРИТИЧНО!)

По умолчанию `vpnc-script` при подключении OpenConnect пишет DNS-серверы VPN в
`/tmp/dnsmasq.d/openconnect.*`, что ломает Podkop.

### Вариант A: Модификация vpnc-script (рекомендуется)

```bash
cp /lib/netifd/vpnc-script /lib/netifd/vpnc-script.bak
```

Отредактировать `/lib/netifd/vpnc-script`, найти строку:

```bash
DNSMASQ_FILE="/tmp/dnsmasq.d/openconnect.$TUNDEV"
```

Заменить на:

```bash
DNSMASQ_FILE="/tmp/openconnect-dns.$TUNDEV"
```

Теперь файл DNS-конфига будет создаваться вне директории dnsmasq, и dnsmasq его не прочитает.

### Вариант B: Кастомный скрипт

Создать `/etc/openconnect/no-dns-vpnc-script`:

```bash
#!/bin/sh
# Запускаем оригинальный скрипт, но перенаправляем DNS-файл
export DNSMASQ_FILE="/tmp/openconnect-dns.$TUNDEV"
exec /lib/netifd/vpnc-script "$@"
```

```bash
chmod +x /etc/openconnect/no-dns-vpnc-script
```

Затем в конфигурации интерфейса указать этот скрипт (см. Шаг 4).

---

## Шаг 4. Настройка интерфейса OpenConnect

### Через UCI (SSH):

```bash
# Создать интерфейс
uci set network.corp_vpn=interface
uci set network.corp_vpn.proto='openconnect'
uci set network.corp_vpn.uri='https://vpn.your-corp.com'
uci set network.corp_vpn.username='ВАШ_ЛОГИН'
uci set network.corp_vpn.password='ВАШ_ПАРОЛЬ'

# Раскомментируйте нужное для 2FA (по результатам Шага 2):
# Если сервер принимает "push" как второй пароль:
# uci set network.corp_vpn.password2='push'

# Хэш сертификата сервера (из Шага 2):
uci set network.corp_vpn.serverhash='sha256:ХЭШИЗШАГА2'

# Если сервер спрашивал GROUP:
# uci set network.corp_vpn.authgroup='ИМЯ_ГРУППЫ'

# КРИТИЧНО: не менять default route и DNS
uci set network.corp_vpn.defaultroute='0'
uci set network.corp_vpn.peerdns='0'

# Кастомный скрипт без DNS (если использовали Вариант Б в Шаге 3):
# uci set network.corp_vpn.script='/etc/openconnect/no-dns-vpnc-script'

# Не запускать автоматически при загрузке (2FA не пройдёт без вас)
uci set network.corp_vpn.auto='0'

uci commit network
```

### Через LuCI:

1. **Network → Interfaces → Add new interface**
2. Name: `corp_vpn`
3. Protocol: **OpenConnect (CISCO AnyConnect)**
4. Заполнить Server, Username, Password, Password2 (если нужно)
5. Во вкладке **Advanced Settings**:
   - Снять галку **Use default gateway** (= `defaultroute '0'`)
   - Снять галку **Use DNS servers advertised by peer** (= `peerdns '0'`)
6. **Firewall Settings** → назначить зону `wan` (или создать отдельную `corp`)
7. Save & Apply

### Первый запуск

```bash
ifup corp_vpn
```

Если 2FA с push — **сразу подтвердите на телефоне!** Таймаут обычно 30-60 секунд.

Проверить статус:

```bash
# Состояние интерфейса
ifstatus corp_vpn

# Есть ли маршруты от VPN
ip route | grep corp_vpn

# Проверить IP через VPN
curl --interface vpn-corp_vpn ifconfig.me
```

Если не подключается — смотреть логи:

```bash
logread | grep openconnect
```

---

## Шаг 5. Настройка Podkop

### 5a. Создать секцию для корпоративного VPN

В LuCI → **Services → Podkop**:

1. Нажать **Add** (новая секция)
2. Имя: `corp` (или любое)
3. **Connection Type**: `VPN`
4. **Network Interface**: выбрать `corp_vpn`

### 5b. Добавить корпоративные домены

В секции `corp`:

1. **User Domain List Type**: Text List
2. Ввести домены (по одному на строку):
   ```
   corp.company.com
   gitlab.company.com
   jira.company.com
   confluence.company.com
   ```

### 5c. Включить Domain Resolver (Split DNS)

Это нужно чтобы корпоративные домены резолвились через корпоративный DNS **внутри VPN-туннеля**.

1. В секции `corp` включить **Domain Resolver**
2. **DNS Protocol Type**: UDP
3. **DNS Server**: IP корпоративного DNS-сервера (узнайте из Шага 2 — переменная `INTERNAL_IP4_DNS`, или спросите у сисадминов; типично `10.x.x.x`)

### 5d. Если нужны корпоративные подсети

Если есть известные подсети (например, `10.0.0.0/8`), добавьте их в **User Subnet List** секции `corp`.

### 5e. Save & Apply

Нажать **Save & Apply** и перезапустить Podkop:

```bash
service podkop restart
```

---

## Шаг 6. Проверка

### DNS работает через Podkop:

```bash
# С клиентского устройства (ПК):
nslookup fakeip.podkop.fyi
# Ожидаемо: 198.18.x.x
```

### Корпоративные ресурсы доступны:

```bash
# С клиентского устройства:
ping gitlab.company.com
# или
curl https://gitlab.company.com
```

### Podkop не сломан:

Открыть заблокированный сайт из списков Podkop — должен работать.

### С роутера:

```bash
# Проверить что sing-box слушает
netstat -tanp | grep sing-box

# Проверить что OpenConnect работает
ifstatus corp_vpn | grep up

# Проверить маршруты
ip route | grep corp_vpn
```

---

## Шаг 7. Ежедневное использование

### Подключение к корп. VPN (ручное, из-за 2FA):

```bash
ifup corp_vpn
# → Подтвердить push на телефоне
```

Или через LuCI: **Network → Interfaces → corp_vpn → Connect**

### Отключение:

```bash
ifdown corp_vpn
```

### Сессия истекла?

Корпоративный VPN рано или поздно разорвёт соединение (политика сервера, обычно 8-24ч). Podkop продолжит работать — просто корпоративные ресурсы станут недоступны. Повторите `ifup corp_vpn` + подтверждение на телефоне.

---

## Обходной путь для сложной 2FA (Duo HTML forms, SAML)

Если Шаг 2 показал ошибку `duo_form` или SAML, стандартный подход не работает.

### Cookie-based подход

**На ПК (Windows)** запустите (нужен установленный openconnect):

```bash
openconnect --authenticate --useragent="AnyConnect" https://vpn.your-corp.com
```

Пройдите интерактивную аутентификацию. После успеха будет выведено:

```
COOKIE=...
HOST=...
CONNECT_URL=...
FINGERPRINT=...
RESOLVE=...
```

Скопируйте COOKIE. На роутере:

```bash
echo "COOKIE_VALUE" | openconnect --cookie-on-stdin https://vpn.your-corp.com \
    --servercert sha256:... \
    --background \
    --script /lib/netifd/vpnc-script \
    --interface vpn-corp_vpn
```

Cookie живёт ограниченное время (часы). Процедуру нужно повторять.

### Скрипт автоматизации для Windows

Можно написать скрипт, который:
1. Получает cookie через `openconnect --authenticate` на ПК
2. Передаёт его на роутер по SSH
3. Роутер подключается с cookie

---

## Что делать если не работает

| Симптом | Причина | Решение |
|---------|---------|---------|
| Podkop сломался после подключения VPN | DNS перезаписан vpnc-script | Проверить Шаг 3. `ls /tmp/dnsmasq.d/` не должен содержать `openconnect.*` |
| OpenConnect не подключается | 2FA не проходит | Проверить логи `logread \| grep openconnect`. Попробовать из CLI |
| Корп. ресурсы недоступны, Podkop работает | Нет маршрута или DNS не резолвит | Проверить `ip route \| grep corp_vpn` и Domain Resolver в Podkop |
| Аккаунт заблокирован | Агрессивный reconnect при 2FA | Убедиться что `auto='0'`, не оставлять интерфейс в состоянии "connecting" |
| `curl --interface vpn-corp_vpn` зависает | Туннель не работает | `ifdown corp_vpn && ifup corp_vpn`, подтвердить 2FA |

---

## Структура итоговой конфигурации

```
┌─────────────────────────────────────────────────┐
│                  Клиентское устройство           │
│          DNS → роутер, Gateway → роутер          │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│                    Роутер (OpenWrt)               │
│                                                   │
│  dnsmasq :53 → sing-box 127.0.0.42:53            │
│                                                   │
│  Podkop секция "main" (proxy/vpn):               │
│    → заблокированные сайты → VLESS/WG/etc        │
│                                                   │
│  Podkop секция "corp" (vpn):                     │
│    → corp.company.com → OpenConnect интерфейс    │
│    → Domain Resolver → корп. DNS через туннель   │
│                                                   │
│  Остальной трафик → напрямую через WAN           │
└──────────────────────────────────────────────────┘
```
