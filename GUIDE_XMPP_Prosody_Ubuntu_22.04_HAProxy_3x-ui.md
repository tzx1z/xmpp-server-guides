# Пошаговое руководство: установка XMPP-сервера Prosody на VPS в России (Ubuntu 22.04)

Версии: Ubuntu 22.04.5 LTS, Prosody 13.0.7 из репозитория packages.prosody.im, coturn 4.5.2 и certbot 1.21 с плагином dns-cloudflare из репозитория Ubuntu. DNS домена обслуживает Cloudflare. Конфигурации Prosody и coturn из руководства проверены на этих версиях.

## Содержание

1. [Введение](#1-введение)
2. [Проверка текущего состояния VPS](#2-проверка-текущего-состояния-vps)
3. [DNS-записи в Cloudflare](#3-dns-записи-в-cloudflare)
4. [Подготовка системы и установка Prosody](#4-подготовка-системы-и-установка-prosody)
5. [Файрвол](#5-файрвол)
6. [SSL-сертификат](#6-ssl-сертификат)
7. [Конфигурация Prosody](#7-конфигурация-prosody)
8. [Звонки: coturn](#8-звонки-coturn)
9. [Запуск и проверка](#9-запуск-и-проверка)
10. [Пользователи и клиенты](#10-пользователи-и-клиенты)
11. [Безопасность и обслуживание](#11-безопасность-и-обслуживание)
12. [Устранение неполадок](#12-устранение-неполадок)
13. [Резюме](#13-резюме)
14. [Приложение А. XMPP и обмен файлами через порт 443](#приложение-а-xmpp-и-обмен-файлами-через-порт-443)

---

## 1. Введение

### 1.1. Что такое XMPP и Prosody

XMPP - открытый протокол обмена сообщениями. Серверы XMPP связаны между собой по тому же принципу, что и почтовые: пользователь сервера `inkov.dev` может переписываться с пользователями любых других XMPP-серверов. Prosody - XMPP-сервер на языке Lua. Он использует десятки мегабайт памяти, настраивается одним файлом и подходит для личного сервера. После выполнения руководства сервер обслуживает адреса вида `user@inkov.dev`, хранит историю сообщений, передает файлы, поддерживает групповые чаты и звонки.

### 1.2. Схема

```text
Cloudflare DNS, зона inkov.dev
├── inkov.dev                    A      GitHub Pages (не меняется)
├── xmpp.inkov.dev               A      <IP_VPS>
├── conference.inkov.dev         CNAME  xmpp.inkov.dev
├── upload.inkov.dev             CNAME  xmpp.inkov.dev
├── turn.inkov.dev               CNAME  xmpp.inkov.dev
├── _xmpp-client._tcp.inkov.dev  SRV    0 5 5222 xmpp.inkov.dev
└── _xmpp-server._tcp.inkov.dev  SRV    0 5 5269 xmpp.inkov.dev

VPS <IP_VPS>, Ubuntu 22.04
├── 3x-ui / Xray   443/tcp, порт панели, inbound-порты   (без изменений)
├── Prosody        5222/tcp          подключение клиентов
│                  5269/tcp          связь с другими серверами
│                  5281/tcp          HTTPS: обмен файлами
│                  5280/tcp          только 127.0.0.1
└── coturn         3478/tcp и udp    STUN/TURN для звонков
                   50000-50100/udp   передача медиа
```

Порядок подключения:

1. Клиент с адресом `user@inkov.dev` запрашивает SRV-запись `_xmpp-client._tcp.inkov.dev` и получает ответ: сервер `xmpp.inkov.dev`, порт 5222.
2. Клиент подключается к VPS, Prosody предъявляет сертификат на `inkov.dev`.
3. Другие XMPP-серверы находят сервер через SRV-запись `_xmpp-server._tcp.inkov.dev`, порт 5269.
4. Сайт на GitHub Pages продолжает работать: у SRV-записей другие имена, они не пересекаются с A-записями `inkov.dev`.

### 1.3. Принятые решения

- **Адреса `user@inkov.dev`, сервер на `xmpp.inkov.dev`.** В Prosody настроен `VirtualHost "inkov.dev"`, SRV-записи связывают домен с сервером.
- **Сертификат на `inkov.dev` и `*.inkov.dev`.** Клиенты и другие серверы проверяют сертификат по домену из адреса (RFC 6120), а не по имени из SRV-записи. Apex-домен `inkov.dev` указывает на GitHub, поэтому проверку через HTTP (HTTP-01) с VPS пройти нельзя. Используется проверка через DNS (DNS-01): certbot сам создает временную TXT-запись через API Cloudflare. Один wildcard-сертификат покрывает `inkov.dev` и все поддомены.
- **Prosody 13 из официального репозитория.** В репозитории Ubuntu 22.04 версия 0.11, в ней нет обмена файлами (`mod_http_file_share`) и поддержки внешнего TURN-сервера (`mod_turn_external`).
- **Порт 443 в основной схеме не используется.** Его занимает Xray, обмен файлами работает по HTTPS на порту 5281. Перенос подключения клиентов и обмена файлами на 443 без расшифровки трафика описан в приложении А.
- **3x-ui не меняется.** Действия на VPS только добавляют пакеты, файлы конфигурации и правила файрвола.
- **Звонки через coturn.** `mod_turn_external` только выдает клиентам адрес и временные учетные данные TURN-сервера. Сам TURN-сервер ставится отдельно (раздел 8).

### 1.4. Ограничения схемы

- Порты 5222 и 5281 нестандартные. В сетях, где разрешен только порт 443 (часть корпоративных сетей), подключение и обмен файлами работать не будут. Вариант для таких сетей - приложение А.
- IP-адрес VPS публикуется в DNS как адрес `xmpp.inkov.dev`. Если 3x-ui использует маскировку под сторонний сайт (например, VLESS Reality), открытые порты XMPP и сертификат `inkov.dev` на том же IP позволяют связать сервер с доменом.
- Домен в адресах пользователей меняется только вместе с учетными записями: при смене домена их нужно создать заново.

### 1.5. Параметры

| Параметр | Значение | Где используется |
|---|---|---|
| XMPP-домен | `inkov.dev` | адреса пользователей, `VirtualHost` |
| Хост сервера | `xmpp.inkov.dev` | A-запись, цель SRV-записей |
| `<IP_VPS>` | публичный IPv4 VPS, в примерах `203.0.113.10` | DNS, проверки |
| `<EMAIL>` | email для аккаунта Let's Encrypt | certbot |
| `<CF_API_TOKEN>` | API-токен Cloudflare | только файл `/root/.secrets/cloudflare.ini` |
| `<SSH_PORT>` | порт SSH, обычно 22 | файрвол |
| `<PANEL_PORT>` | порт панели 3x-ui | файрвол |
| `<TURN_SECRET>` | общий секрет Prosody и coturn | создается в разделе 7.2, подставляется командой |

Команды в блоках кода выполняются на VPS, если не указано другое. Плейсхолдеры в угловых скобках замените своими значениями.

### 1.6. Что понадобится

- VPS с Ubuntu 22.04, доступ по SSH, пользователь с правами sudo.
- Доступ к панели Cloudflare для домена `inkov.dev`.
- Доступ к консоли VPS в панели хостера (VNC или аналог) на случай ошибки в настройке файрвола.
- XMPP-клиент на телефоне или компьютере (раздел 10).

---

## 2. Проверка текущего состояния VPS

Цель раздела: собрать данные для настройки и убедиться, что порты XMPP свободны. Команды раздела только читают состояние системы, кроме резервной копии в 2.6.

### 2.1. ОС и ресурсы

```bash
# Версия ОС: ожидается Ubuntu 22.04.x LTS
lsb_release -ds
# Свободное место на корневом разделе
df -h /
# Свободная память
free -h
```

**Результат:** `Ubuntu 22.04.5 LTS`, свободно не меньше 6 ГБ на диске: до 5 ГБ отводится под файлы пользователей (раздел 7). Если места меньше, уменьшите `http_file_share_global_quota` в конфигурации Prosody. Prosody и coturn вместе используют меньше 100 МБ памяти.

### 2.2. Время

```bash
timedatectl
```

**Результат:** `System clock synchronized: yes` и `NTP service: active`. Неверное время приводит к ошибкам проверки сертификатов. Если синхронизация выключена:

```bash
sudo timedatectl set-ntp true
```

На VPS с виртуализацией OpenVZ или LXC временем управляет хостер: при расхождении обратитесь в поддержку.

### 2.3. Занятые порты и порты 3x-ui

```bash
# Все прослушиваемые порты с именами процессов
sudo ss -tulpn
# Порты 3x-ui и Xray: понадобятся в разделе 5
sudo ss -tulpn | grep -E 'x-ui|xray'
# Порт SSH
sudo ss -tlpn | grep sshd
```

Пример вывода второй команды:

```text
tcp   LISTEN 0  4096   *:443    *:*   users:(("xray-linux-amd6",pid=812,fd=7))
tcp   LISTEN 0  4096   *:2053   *:*   users:(("x-ui",pid=790,fd=9))
```

Здесь 443 - inbound Xray, 2053 - панель 3x-ui. Запишите все порты из вывода вместе с протоколом (`tcp` или `udp`).

Проверьте, что порты Prosody и coturn свободны:

```bash
sudo ss -tulpn | grep -E ':(5222|5269|5280|5281|5000|3478|5349)\b'
sudo ss -tulpn | grep -E ':50(0[0-9]{2}|100)\b'
```

**Результат:** вывод пустой. Если порт занят процессом 3x-ui, 3x-ui не меняйте, а замените порт в этом руководстве: порт 5281 и диапазон coturn меняются только в конфигурации, порты 5222 и 5269 - в конфигурации и в SRV-записях.

### 2.4. Файрвол и сертификат панели 3x-ui

```bash
sudo ufw status verbose
```

- `Status: active` - ufw включен, в разделе 5 нужно только добавить правила.
- `Status: inactive` - ufw выключен, в разделе 5 описано безопасное включение.

Если ufw неактивен, проверьте, нет ли других правил фильтрации:

```bash
sudo iptables -S | head -20
```

Строки `-P INPUT ACCEPT`, `-P FORWARD ACCEPT`, `-P OUTPUT ACCEPT` без других правил означают, что фильтрации нет.

Проверьте, получен ли сертификат панели 3x-ui через меню `x-ui`:

```bash
sudo ls /root/.acme.sh 2>/dev/null
```

Если в выводе есть каталоги с именами доменов, сертификат панели продлевает acme.sh. Если сертификат выпускался в режиме standalone, acme.sh использует для продления порт 80, и этот порт нужно оставить открытым (раздел 5).

Также проверьте в панели хостера, есть ли внешний (облачный) файрвол. Если он включен, порты из раздела 5 нужно открыть и в нем.

### 2.5. IP-адреса и IPv6

```bash
# Адреса на сетевых интерфейсах
ip -br addr
# Публичный IPv4, который видят внешние серверы
curl -4 -s https://ipv4-internet.yandex.net/api/v0/ip; echo
# Публичный IPv6: ошибка или пустой ответ означает, что IPv6 не работает
curl -6 -s --max-time 5 https://ipv6-internet.yandex.net/api/v0/ip; echo
```

- Публичный IPv4 из второй команды есть в выводе `ip -br addr` - VPS получает адрес напрямую. Это обычный случай.
- На интерфейсе только частный адрес (`10.x.x.x`, `172.16-31.x.x`, `192.168.x.x`) - хостер использует NAT. Для coturn тогда нужен параметр `external-ip` (раздел 8.2).
- Третья команда вернула адрес - IPv6 работает, для `xmpp.inkov.dev` можно создать AAAA-запись. Если адреса нет, AAAA-запись не создавайте.

### 2.6. Резервная копия базы 3x-ui

Руководство не меняет 3x-ui. Копия нужна на случай ошибки при настройке файрвола или системы.

```bash
sudo cp /etc/x-ui/x-ui.db /root/x-ui.db.bak-$(date +%F)
sudo ls -l /root/x-ui.db.bak-*
```

---

## 3. DNS-записи в Cloudflare

Где выполняется: панель Cloudflare в браузере. Проверка - на VPS или на локальном компьютере.

### 3.1. Создание записей

1. Откройте https://dash.cloudflare.com и выберите домен `inkov.dev`.
2. Перейдите в **DNS → Records**.
3. Для каждой строки таблицы нажмите **Add record**, заполните поля и нажмите **Save**.

| Type | Name | Значение | Proxy status | TTL |
|---|---|---|---|---|
| A | `xmpp` | IPv4 address: `<IP_VPS>` | DNS only | Auto |
| CNAME | `conference` | Target: `xmpp.inkov.dev` | DNS only | Auto |
| CNAME | `upload` | Target: `xmpp.inkov.dev` | DNS only | Auto |
| CNAME | `turn` | Target: `xmpp.inkov.dev` | DNS only | Auto |
| SRV | `_xmpp-client._tcp` | Priority `0`, Weight `5`, Port `5222`, Target `xmpp.inkov.dev` | нет | Auto |
| SRV | `_xmpp-server._tcp` | Priority `0`, Weight `5`, Port `5269`, Target `xmpp.inkov.dev` | нет | Auto |
| AAAA, только при рабочем IPv6 | `xmpp` | IPv6 address: адрес VPS | DNS only | Auto |
| CNAME, только для Proxy65 | `proxy` | Target: `xmpp.inkov.dev` | DNS only | Auto |

> **Важно:** для записей A, AAAA и CNAME Cloudflare по умолчанию включает проксирование (Proxy status: Proxied, оранжевое облако). Переключите его в **DNS only** (серое облако). Прокси Cloudflare пропускает только HTTP и HTTPS на ограниченном наборе портов, XMPP и TURN через него не работают.

Пояснения к полям:

- **Name** вводится относительно зоны: `xmpp` означает `xmpp.inkov.dev`, `_xmpp-client._tcp` означает `_xmpp-client._tcp.inkov.dev`. Имя зоны Cloudflare добавит сам.
- SRV-записи создаются в зоне `inkov.dev`, а не на поддомене: клиенты ищут их по домену из адреса пользователя. Если форма SRV показывает отдельные поля Service и Protocol, заполните Service `_xmpp-client` (или `_xmpp-server`), Protocol `TCP`, Name `@`.
- **Target** SRV-записи - имя с A-записью (`xmpp.inkov.dev`), а не CNAME: так требует RFC 2782.
- **Priority** задает порядок перебора серверов, **Weight** - распределение нагрузки между серверами с одинаковым приоритетом. Для одного сервера подходят любые значения, `0` и `5` - распространенный вариант.
- **TTL Auto** в Cloudflare равен 300 секундам.
- Для записей в режиме DNS only панель может предупредить, что IP-адрес будет виден. Для этой схемы это ожидаемо.
- Существующие A-записи `inkov.dev` (GitHub Pages) и запись `www`, если она есть, не меняйте.

### 3.2. CAA-записи

CAA-записи ограничивают список центров сертификации, которые могут выпускать сертификаты для домена.

```bash
# Если команда dig не найдена: sudo apt install -y dnsutils
dig +short CAA inkov.dev
```

- Пустой вывод - ограничений нет, дальше ничего делать не нужно.
- Если записи есть, среди них должна быть `0 issue "letsencrypt.org"`. Если есть записи с тегом `issuewild`, нужна и `0 issuewild "letsencrypt.org"`, иначе wildcard-сертификат не будет выпущен. Недостающую запись добавьте в Cloudflare: Type `CAA`, Name `@`, тег выбирается в поле **Tag**, значение `letsencrypt.org`.

GitHub Pages тоже получает сертификаты Let's Encrypt, поэтому существующие CAA-записи обычно уже разрешают `letsencrypt.org`.

### 3.3. Проверка

Запросы напрямую к DNS-серверу Cloudflare показывают записи сразу после сохранения, без кэша промежуточных резолверов.

```bash
# DNS-сервер Cloudflare для зоны
NS=$(dig +short NS inkov.dev | head -1); echo "$NS"
dig +short @"$NS" A xmpp.inkov.dev
dig +short @"$NS" SRV _xmpp-client._tcp.inkov.dev
dig +short @"$NS" SRV _xmpp-server._tcp.inkov.dev
dig +short @"$NS" upload.inkov.dev
```

**Результат** (с адресом `203.0.113.10` из примера):

```text
xxxx.ns.cloudflare.com.
203.0.113.10
0 5 5222 xmpp.inkov.dev.
0 5 5269 xmpp.inkov.dev.
xmpp.inkov.dev.
203.0.113.10
```

Затем проверьте ответ обычного резолвера:

```bash
dig +short SRV _xmpp-client._tcp.inkov.dev
```

Ответ должен совпасть с предыдущим. Если он пустой, подождите: резолвер, который уже запрашивал это имя до создания записи, хранит отрицательный ответ до 30 минут.

---

## 4. Подготовка системы и установка Prosody

### 4.1. Зеркала APT

```bash
sudo apt update
```

**Результат:** строки `Hit:` и `Get:` без `Err:` и без зависаний. В этом случае зеркала менять не нужно: у российских хостеров они часто уже локальные.

Если `apt update` завершается с ошибками `Err:` или зависает, переключите систему на зеркало Яндекса:

```bash
# Резервная копия текущего списка
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
# Новый список: основной репозиторий, обновления, backports и обновления безопасности
sudo tee /etc/apt/sources.list > /dev/null <<'EOF'
deb https://mirror.yandex.ru/ubuntu jammy main restricted universe multiverse
deb https://mirror.yandex.ru/ubuntu jammy-updates main restricted universe multiverse
deb https://mirror.yandex.ru/ubuntu jammy-backports main restricted universe multiverse
deb https://mirror.yandex.ru/ubuntu jammy-security main restricted universe multiverse
EOF
sudo apt update
```

Откат:

```bash
sudo cp /etc/apt/sources.list.bak /etc/apt/sources.list && sudo apt update
```

Основной список репозиториев Ubuntu 22.04 хранится в `/etc/apt/sources.list`. Файл `/etc/apt/sources.list.d/ubuntu.sources` появился только в Ubuntu 24.04. Если ошибку выдает дополнительный репозиторий, найдите его файл в `ls /etc/apt/sources.list.d/`.

### 4.2. Обновление системы

```bash
sudo apt upgrade -y
# Нужна ли перезагрузка после обновления ядра
ls /var/run/reboot-required 2>/dev/null && echo "нужна перезагрузка"
```

Если нужна перезагрузка, выполните `sudo reboot` в удобное время: во время перезагрузки VPN 3x-ui недоступен. Команду `do-release-upgrade` не запускайте: руководство рассчитано на Ubuntu 22.04.

### 4.3. Репозиторий Prosody

```bash
# Доступность репозитория с VPS: ожидается 200
curl -s -o /dev/null -w '%{http_code}\n' --max-time 15 http://packages.prosody.im/debian/dists/jammy/Release
# Подключение репозитория. Файл содержит адрес репозитория и ключ подписи (Signed-By),
# отдельный ключ и apt-key не нужны
sudo wget https://prosody.im/downloads/repos/jammy/prosody.sources -O /etc/apt/sources.list.d/prosody.sources
sudo apt update
# Версия, которая будет установлена
apt-cache policy prosody
```

**Результат** `apt-cache policy prosody`:

```text
prosody:
  Installed: (none)
  Candidate: 13.0.7-1~jammy2
  Version table:
     13.0.7-1~jammy2 500
        500 http://packages.prosody.im/debian jammy/main amd64 Packages
     0.11.13-1 500
        500 http://archive.ubuntu.com/ubuntu jammy/universe amd64 Packages
```

Номер версии может быть новее. Важно, чтобы `Candidate` был из `packages.prosody.im` и не ниже 13.0.

Если репозиторий недоступен (curl возвращает `000` или завершается по таймауту), скачайте пакет `prosody_13.*~jammy*_amd64.deb` на другом компьютере из https://packages.prosody.im/debian/pool/main/p/prosody/, перенесите на VPS через `scp` и установите командой `sudo apt install --no-install-recommends ./prosody_*.deb lua-unbound lua-readline`. Версию 0.11 из репозитория Ubuntu не устанавливайте.

### 4.4. Установка Prosody

```bash
sudo apt install --no-install-recommends prosody lua-unbound lua-readline
```

> **Важно:** флаг `--no-install-recommends` обязателен. Без него apt установит рекомендуемый пакет `luarocks`. На Ubuntu 22.04 он зависит от Lua 5.1 и переключает на нее команду `lua`, после чего Prosody не запускается с ошибкой `Prosody is no longer compatible with Lua 5.1`. По той же причине не устанавливайте вручную `luarocks`, `lua5.4` и `liblua5.4-dev`: нужная версия Lua 5.4 ставится как зависимость пакета.

`lua-unbound` (DNS-резолвер) и `lua-readline` (редактирование строк в `prosodyctl shell`) рекомендованы Prosody и не тянут лишних зависимостей.

Проверка:

```bash
sudo prosodyctl about | grep -E '^Prosody|Lua version|luaunbound'
systemctl status prosody --no-pager
```

**Результат:**

```text
Prosody 13.0.7
Lua version:             	Lua 5.4
luaunbound:   	1.0.0
```

и `Active: active (running)`. Сейчас Prosody работает с конфигурацией по умолчанию (хост `localhost`), она заменяется в разделе 7.

---

## 5. Файрвол

### 5.1. Правила

> **Важно:** если ufw сейчас неактивен, его включение без правил для SSH и 3x-ui закроет доступ к серверу и остановит VPN. Добавьте все правила до `ufw enable`. Не закрывайте текущую SSH-сессию, пока не проверите новое подключение.

```bash
# 1. SSH и 3x-ui: порты из раздела 2.3
sudo ufw allow <SSH_PORT>/tcp comment 'SSH'
sudo ufw allow <PANEL_PORT>/tcp comment '3x-ui panel'
sudo ufw allow 443/tcp comment 'Xray'
# Остальные inbound-порты Xray по той же схеме, для UDP: sudo ufw allow <PORT>/udp
# Если в разделе 2.4 найден каталог /root/.acme.sh:
# sudo ufw allow 80/tcp comment 'acme.sh 3x-ui'

# 2. XMPP
sudo ufw allow 5222/tcp comment 'XMPP clients'
sudo ufw allow 5269/tcp comment 'XMPP servers'
sudo ufw allow 5281/tcp comment 'XMPP HTTPS file share'

# 3. Список добавленных правил
sudo ufw show added
```

**Результат** `ufw show added`: список всех правил, включая SSH и порты 3x-ui. Если ufw уже был активен, правила действуют сразу, переходите к 5.3.

Порт 5280 не открывается: он нужен только локально. Порты coturn (3478 и 50000-50100/udp) открываются в разделе 8.3, после настройки coturn. Порты 5000 (Proxy65) и 5349 (TURN с TLS) открываются только при включении этих функций (разделы 7.3 и 8.4).

### 5.2. Включение ufw (только если он был неактивен)

```bash
sudo ufw enable
sudo ufw status verbose
```

`ufw enable` предупредит о возможном разрыве SSH-соединений, подтвердите `y`. Затем, не закрывая текущую сессию, откройте новое SSH-подключение и проверьте работу VPN. Если доступ пропал, в консоли хостера выполните `sudo ufw disable`.

**Результат:**

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
...
5222/tcp                   ALLOW IN    Anywhere                   # XMPP clients
```

Правила действуют для IPv4 и IPv6: в `/etc/default/ufw` по умолчанию `IPV6=yes`.

> **Важно:** если кроме 3x-ui на VPS работает VPN с маршрутизацией на уровне ядра (WireGuard, OpenVPN, AmneziaWG), включенный ufw блокирует пересылку трафика (`DEFAULT_FORWARD_POLICY="DROP"` в `/etc/default/ufw`). В этом случае настройте пересылку отдельно или оставьте ufw выключенным. Xray в 3x-ui работает как прокси и пересылку не использует.

### 5.3. Файрвол хостера

Если в разделе 2.4 найден внешний файрвол в панели хостера, откройте в нем те же порты: 5222/tcp, 5269/tcp, 5281/tcp. Порты coturn (3478/tcp и udp, 50000-50100/udp) откройте там же после раздела 8.3.

---

## 6. SSL-сертификат

### 6.1. Доступность Let's Encrypt

```bash
curl -s -o /dev/null -w '%{http_code}\n' --max-time 15 https://acme-v02.api.letsencrypt.org/directory
```

**Результат:** `200`. Доступность Let's Encrypt из России зависит от сети хостера, поэтому руководство опирается на эту проверку. Если результат `000` или другой код, перейдите к запасным вариантам в 6.8.

### 6.2. API-токен Cloudflare

certbot создает временную TXT-запись `_acme-challenge.inkov.dev` через API Cloudflare. Для этого нужен токен с правом редактирования DNS только в зоне `inkov.dev`.

Где выполняется: панель Cloudflare.

1. Откройте **My Profile → API Tokens** (https://dash.cloudflare.com/profile/api-tokens) и нажмите **Create Token**.
2. Выберите шаблон **Edit zone DNS** и нажмите **Use template**.
3. **Permissions**: оставьте `Zone` / `DNS` / `Edit`.
4. **Zone Resources**: `Include` / `Specific zone` / `inkov.dev`.
5. **Client IP Address Filtering** (рекомендуется): `Is in` и адрес `<IP_VPS>`. Если на VPS работает IPv6, добавьте и IPv6-адрес: запросы к API могут уходить по IPv6.
6. Нажмите **Continue to summary**, затем **Create Token**.
7. Скопируйте токен. Cloudflare показывает его один раз.

### 6.3. Токен на VPS

```bash
# Каталог и пустой файл, доступные только root
sudo install -d -m 700 /root/.secrets
sudo install -m 600 /dev/null /root/.secrets/cloudflare.ini
sudo nano /root/.secrets/cloudflare.ini
```

Содержимое файла:

```ini
# API-токен Cloudflare: Zone / DNS / Edit, только зона inkov.dev
dns_cloudflare_api_token = <CF_API_TOKEN>
```

Токен вставляется в редакторе и не попадает в историю shell.

Проверка токена и доступности API Cloudflare с VPS (токен читается из файла):

```bash
sudo sh -c 'curl -s --max-time 20 https://api.cloudflare.com/client/v4/user/tokens/verify -H "Authorization: Bearer $(sed -n "s/^dns_cloudflare_api_token *= *//p" /root/.secrets/cloudflare.ini)"'; echo
```

**Результат:**

```text
{"result":{"id":"...","status":"active"},"success":true,"errors":[],"messages":[{"code":10000,"message":"This API Token is valid and active","type":null}]}
```

- `"success":false` - токен неверный или скопирован не полностью.
- Команда завершается по таймауту или возвращает пустой ответ - API Cloudflare недоступен из сети хостера. С июня 2025 года российские провайдеры ограничивают соединения с сетью Cloudflare. В этом случае используйте ручной вариант из 6.8.

### 6.4. Установка certbot

```bash
sudo apt install certbot python3-certbot-dns-cloudflare
certbot --version
```

**Результат:** `certbot 1.21.0`. Это версия из репозитория Ubuntu. Она старше актуальной, но поддерживает DNS-01 через API Cloudflare с токеном. Snap-версию certbot параллельно не ставьте: обе установки используют каталог `/etc/letsencrypt` и запускают продление независимо друг от друга.

### 6.5. Пробный выпуск

```bash
sudo certbot certonly --dry-run \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 60 \
  -d inkov.dev -d '*.inkov.dev' \
  --email <EMAIL> --agree-tos --no-eff-email
```

Параметры:

- `--dry-run` - выпуск на тестовом сервере Let's Encrypt без сохранения сертификата. Проверяет токен, DNS и доступность Let's Encrypt и не расходует лимит: не больше 5 одинаковых сертификатов в неделю.
- `--dns-cloudflare-propagation-seconds 60` - пауза перед проверкой, чтобы TXT-запись появилась на всех DNS-серверах Cloudflare.
- `-d inkov.dev -d '*.inkov.dev'` - один сертификат на домен и все поддомены. Кавычки вокруг `*.inkov.dev` обязательны, иначе shell попытается подставить имена файлов.

**Результат:** `The dry run was successful.` Выполнение занимает около минуты.

### 6.6. Выпуск сертификата

```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 60 \
  -d inkov.dev -d '*.inkov.dev' \
  --email <EMAIL> --agree-tos --no-eff-email \
  --deploy-hook 'prosodyctl --root cert import /etc/letsencrypt/live'
```

`--deploy-hook` выполняется после каждого успешного выпуска и продления. `prosodyctl --root cert import` копирует сертификат и ключ в `/etc/prosody/certs` с владельцем `prosody` и перезагружает Prosody. Указывать Prosody пути к файлам в `/etc/letsencrypt` напрямую нельзя: ключ там доступен только root, а Prosody работает от пользователя `prosody`.

**Результат:** в выводе есть строки

```text
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/inkov.dev/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/inkov.dev/privkey.pem
```

и сообщение команды импорта `Imported certificate and key for hosts inkov.dev, *.inkov.dev`.

Проверка:

```bash
sudo certbot certificates
sudo ls -l /etc/prosody/certs/
```

**Результат:** `Domains: inkov.dev *.inkov.dev`, `Expiry Date` примерно через 90 дней. В `/etc/prosody/certs` есть файлы `inkov.dev.crt` и `inkov.dev.key` с владельцем `prosody`.

### 6.7. Автоматическое продление

certbot из пакета Ubuntu устанавливает таймер systemd, который дважды в сутки запускает `certbot renew`. Сертификат продлевается за 30 дней до окончания срока, после продления выполняется `--deploy-hook`.

```bash
# Таймер продления
systemctl list-timers certbot.timer --no-pager
# Команда импорта сохранена в настройках продления
sudo grep renew_hook /etc/letsencrypt/renewal/inkov.dev.conf
# Проверка продления на тестовом сервере
sudo certbot renew --dry-run
```

**Результат:** `certbot.timer` в списке со временем следующего запуска; строка `renew_hook = prosodyctl --root cert import /etc/letsencrypt/live`; сообщение `Congratulations, all simulated renewals succeeded`.

При `--dry-run` deploy hook не выполняется. Команда импорта проверяется отдельно в разделе 9.2.

> **Важно:** с июня 2025 года Let's Encrypt не рассылает письма об истечении сертификатов. Если продление перестанет работать (например, API Cloudflare станет недоступен с VPS), уведомления не будет. Раз в месяц проверяйте срок командой `sudo certbot certificates` или настройте внешний мониторинг сертификата по адресу `https://upload.inkov.dev:5281`.

### 6.8. Запасные варианты

**Вариант 1. Ручная проверка DNS-01.** Подходит, если API Cloudflare недоступен с VPS. TXT-записи создаются вручную в панели Cloudflare, автоматического продления нет.

```bash
sudo certbot certonly --manual --preferred-challenges dns \
  -d inkov.dev -d '*.inkov.dev' \
  --email <EMAIL> --agree-tos --no-eff-email \
  --deploy-hook 'prosodyctl --root cert import /etc/letsencrypt/live'
```

certbot по очереди выведет два значения для имени `_acme-challenge.inkov.dev`, по одному на каждое имя в сертификате. Для каждого значения создайте в Cloudflare запись: Type `TXT`, Name `_acme-challenge`, Content - значение из вывода. Перед последним нажатием Enter убедитесь, что видны обе записи:

```bash
dig +short @"$(dig +short NS inkov.dev | head -1)" TXT _acme-challenge.inkov.dev
```

После выпуска удалите TXT-записи. `certbot renew` для такого сертификата не работает: команду нужно повторять вручную до истечения срока, примерно раз в 60 дней.

**Вариант 2. Другой центр сертификации.** Подходит, если недоступен Let's Encrypt, а API Cloudflare работает. ZeroSSL поддерживает ACME и wildcard-сертификаты, но требует ключи EAB из личного кабинета ZeroSSL. К команде из 6.6 добавляются параметры:

```bash
  --server https://acme.zerossl.com/v2/DV90 \
  --eab-kid <EAB_KID> --eab-hmac-key <EAB_HMAC_KEY>
```

**Вариант 3. Самоподписанный сертификат.** Только для проверки своего клиента: федерация с другими серверами работать не будет, клиенты покажут предупреждение.

```bash
sudo prosodyctl cert generate inkov.dev
```

---

## 7. Конфигурация Prosody

### 7.1. Резервная копия

```bash
sudo cp /etc/prosody/prosody.cfg.lua /etc/prosody/prosody.cfg.lua.orig
```

Откат: `sudo cp /etc/prosody/prosody.cfg.lua.orig /etc/prosody/prosody.cfg.lua && sudo systemctl restart prosody`.

В пакете Prosody 13 вся конфигурация хранится в одном файле `/etc/prosody/prosody.cfg.lua`. Каталоги `conf.avail` и `conf.d` из старых пакетов Debian не используются.

### 7.2. Секрет для TURN

```bash
sudo install -d -m 700 /root/.secrets
sudo sh -c 'umask 077; openssl rand -hex 32 > /root/.secrets/turn_secret'
```

Секрет подставляется в конфигурации Prosody (7.4) и coturn (8.2) командой `sed`, копировать его вручную не нужно.

### 7.3. Назначение настроек

| Параметр | Значение | Назначение |
|---|---|---|
| `admins` | `admin@inkov.dev` | администратор сервера: команды из клиента, приглашения |
| `modules_enabled` | список из пакета, плюс `mam` и `turn_external` | набор функций сервера, пояснения ниже |
| `allow_registration` | `false` | регистрация из клиентов закрыта. Учетные записи создает администратор или они создаются по приглашению (модули `invites*`) |
| `c2s_require_encryption`, `s2s_require_encryption` | `true` | подключение без TLS невозможно |
| `s2s_secure_auth` | `true` | другие серверы должны предъявить действительный сертификат |
| `limits` | 10 и 30 КБ/с | ограничение входящего трафика от клиентов и серверов, значения из пакета |
| `pidfile` | `/var/run/prosody/prosody.pid` | по нему `prosodyctl` перезагружает сервер, в том числе после импорта сертификата |
| `authentication` | `internal_hashed` | пароли хранятся в виде хешей SCRAM |
| хранилище | по умолчанию `internal` | файлы в `/var/lib/prosody`. SQLite имеет смысл при десятках пользователей и больших архивах |
| `archive_expires_after` | `1y` | история хранится на сервере год и синхронизируется между устройствами. Меньший срок - меньше данных на сервере. Примеры значений: `1w`, `30d`, `6 months`, `1y`, `never`. Значение `1m` неоднозначно (месяц или минута), Prosody 13 записывает его в журнал как ошибку |
| `http_interfaces` | `127.0.0.1` | нешифрованный HTTP на порту 5280 доступен только с самого сервера |
| `https_ports` | `5281` | HTTPS для обмена файлами |
| `turn_external_*` | `turn.inkov.dev`, секрет, TCP | клиенты получают адрес coturn и временные учетные данные, действующие сутки |
| `certificates` | `certs` | Prosody сам находит сертификат в `/etc/prosody/certs`, для поддоменов - через wildcard |

Модули, от которых зависит работа мобильных клиентов:

- `smacks` - восстановление сессии после смены сети без потери сообщений;
- `carbons` - копии сообщений на все устройства пользователя;
- `csi_simple` - отложенная доставка второстепенного трафика, пока приложение в фоне;
- `cloud_notify` - push-уведомления (XEP-0357), в Prosody 13 входит в поставку и включен по умолчанию;
- `mam` - архив сообщений на сервере.

Компоненты:

- `conference.inkov.dev` (`muc`) - групповые чаты. `muc_mam` хранит историю комнат, `restrict_room_creation = "local"` разрешает создавать комнаты только пользователям `inkov.dev`.
- `upload.inkov.dev` (`http_file_share`) - обмен файлами. Это отдельный компонент, а не модуль из `modules_enabled`. Лимиты: файл до 100 МБ, до 1 ГБ в сутки на пользователя, до 5 ГБ на сервере, хранение 30 дней. Большие файлы записываются на диск потоком, параметр `http_max_content_size` менять не нужно.
- `proxy.inkov.dev` (`proxy65`) - прямая передача файлов для старых клиентов. Современные клиенты используют `http_file_share`, поэтому компонент закомментирован. Для включения уберите `--` в двух строках, создайте DNS-запись `proxy` (раздел 3.1) и откройте порт: `sudo ufw allow 5000/tcp comment 'XMPP proxy65'`.

### 7.4. Файл конфигурации

Скопируйте блок целиком и выполните его в терминале: команда заменит содержимое файла.

```bash
sudo tee /etc/prosody/prosody.cfg.lua > /dev/null <<'EOF'
-- /etc/prosody/prosody.cfg.lua
-- Личный XMPP-сервер для адресов вида user@inkov.dev.
-- Основа - конфигурация из пакета Prosody 13.0.
-- После каждой правки: sudo prosodyctl check config

---------- Глобальные настройки ----------
-- Действуют на весь сервер. Должны стоять выше первой строки VirtualHost или Component.

-- Администраторы. Учетную запись нужно создать отдельно командой prosodyctl adduser.
admins = { "admin@inkov.dev" }

modules_enabled = {
    -- Обязательные
        "disco"; -- обнаружение возможностей сервера
        "roster"; -- список контактов
        "saslauth"; -- аутентификация
        "tls"; -- шифрование соединений

    -- Рекомендуемые
        "blocklist"; -- блокировка пользователей
        "bookmarks"; -- синхронизация списка групповых чатов между клиентами
        "carbons"; -- копии сообщений на все устройства пользователя
        "dialback"; -- запасной способ проверки других серверов через DNS
        "limits"; -- ограничение скорости входящих соединений
        "pep"; -- данные учетной записи: аватары, ключи OMEMO
        "private"; -- старое хранилище настроек клиентов (XEP-0049)
        "smacks"; -- восстановление соединения без потери сообщений (XEP-0198)
        "vcard4"; -- профили пользователей
        "vcard_legacy"; -- совместимость со старым форматом профилей

    -- Мобильные клиенты и удобство
        "account_activity"; -- время последнего входа в учетную запись
        "cloud_notify"; -- push-уведомления для мобильных клиентов (XEP-0357)
        "csi_simple"; -- экономия трафика и заряда батареи на телефонах
        "invites"; -- приглашения
        "invites_adhoc"; -- создание приглашений из клиента
        "invites_register"; -- регистрация по приглашению при закрытой регистрации
        "ping"; -- ответы на XMPP ping
        "register"; -- смена пароля из клиента; регистрацию закрывает allow_registration
        "time"; -- время сервера
        "uptime"; -- время работы сервера
        "version"; -- версия сервера
        "mam"; -- архив сообщений на сервере (XEP-0313)
        "turn_external"; -- данные TURN-сервера для звонков (XEP-0215)

    -- Администрирование
        "admin_adhoc"; -- команды администратора из клиента
        "admin_shell"; -- консоль sudo prosodyctl shell
}

-- Регистрация из клиентов запрещена. Учетные записи создает администратор
-- или они создаются по приглашению.
allow_registration = false

-- Шифрование обязательно для клиентов и серверов
c2s_require_encryption = true
s2s_require_encryption = true

-- Другие серверы должны предъявлять действительный сертификат
s2s_secure_auth = true

-- Ограничение скорости входящего трафика (значения из конфигурации пакета)
limits = {
    c2s = {
        rate = "10kb/s";
    };
    s2sin = {
        rate = "30kb/s";
    };
}

-- Нужен prosodyctl: по этому файлу он находит процесс для перезагрузки,
-- в том числе после обновления сертификата
pidfile = "/var/run/prosody/prosody.pid"

-- Пароли хранятся в виде хешей (SCRAM)
authentication = "internal_hashed"

-- Хранилище по умолчанию: файлы в /var/lib/prosody. Для личного сервера этого достаточно.
--storage = "internal"

-- Срок хранения архива личных сообщений
archive_expires_after = "1y"

-- HTTP. Нешифрованный HTTP (5280) доступен только с самого сервера.
-- HTTPS для обмена файлами - порт 5281, потому что 443 занят Xray.
http_ports = { 5280 }
http_interfaces = { "127.0.0.1" }
https_ports = { 5281 }

-- TURN-сервер для звонков (coturn). Секрет совпадает с static-auth-secret в /etc/turnserver.conf.
-- Если звонки не нужны, удалите эти три строки и "turn_external" из modules_enabled.
turn_external_host = "turn.inkov.dev"
turn_external_secret = "<TURN_SECRET>"
turn_external_tcp = true

-- Журналы
log = {
    info = "/var/log/prosody/prosody.log"; -- замените info на debug для подробного журнала
    error = "/var/log/prosody/prosody.err";
}

-- Каталог сертификатов относительно этого файла: /etc/prosody/certs
certificates = "certs"

----------- Виртуальный хост -----------
-- Домен в адресах пользователей: user@inkov.dev
VirtualHost "inkov.dev"

------------- Компоненты -------------

-- Групповые чаты: комнаты вида room@conference.inkov.dev
Component "conference.inkov.dev" "muc"
    modules_enabled = { "muc_mam" } -- архив сообщений в комнатах
    restrict_room_creation = "local" -- создавать комнаты могут только пользователи inkov.dev
    muc_log_expires_after = "1y" -- срок хранения архива комнат

-- Обмен файлами: https://upload.inkov.dev:5281/
Component "upload.inkov.dev" "http_file_share"
    http_file_share_size_limit = 100 * 1024 * 1024 -- максимальный размер файла: 100 МБ
    http_file_share_daily_quota = 1024 * 1024 * 1024 -- на пользователя в сутки: 1 ГБ
    http_file_share_global_quota = 5 * 1024 * 1024 * 1024 -- всего на сервере: 5 ГБ
    http_file_share_expires_after = "30d" -- файлы удаляются через 30 дней

-- Proxy65 (опционально): прямая передача файлов для старых клиентов.
-- Чтобы включить: уберите "--" в двух строках ниже, создайте DNS-запись proxy
-- и откройте порт 5000/tcp.
--Component "proxy.inkov.dev" "proxy65"
--    proxy65_acl = { "inkov.dev" }
EOF
```

Подстановка секрета, права доступа и проверка:

```bash
# Секрет TURN из файла вместо <TURN_SECRET>
sudo sh -c 'sed -i "s/<TURN_SECRET>/$(cat /root/.secrets/turn_secret)/" /etc/prosody/prosody.cfg.lua'
# Плейсхолдеров не осталось: ожидается 0
sudo grep -c '<TURN_SECRET>' /etc/prosody/prosody.cfg.lua
# Владелец root, группа prosody, остальным доступа нет
sudo chown root:prosody /etc/prosody/prosody.cfg.lua
sudo chmod 640 /etc/prosody/prosody.cfg.lua
# Проверка синтаксиса и параметров
sudo prosodyctl check config
```

**Результат** `check config`:

```text
Checking config...
    The following configuration files have been loaded:
      -  /etc/prosody/prosody.cfg.lua

    Some of your hosts may be missing features due to a lack of configuration.
    For more details, use the 'prosodyctl check features' command.
Done.

All checks passed, congratulations!
```

Сообщение про `check features` относится к подключениям из браузера (раздел 9.3) и на работу сервера не влияет. Новая конфигурация применяется при перезапуске в разделе 9.1.

---

## 8. Звонки: coturn

Если звонки не нужны, пропустите раздел. Тогда удалите из `/etc/prosody/prosody.cfg.lua` строку `"turn_external";` и три строки `turn_external_*`.

### 8.1. Установка

```bash
sudo apt install coturn
# Служба запускается сразу после установки с файлом конфигурации из пакета.
# До настройки ее нужно остановить.
sudo systemctl stop coturn
```

> **Важно:** с файлом конфигурации из пакета coturn выделяет relay-порты без проверки учетных данных. Поэтому служба останавливается сразу после установки, а порты TURN в файрволе открываются только после настройки (раздел 8.3).

### 8.2. Конфигурация

```bash
# Резервная копия файла из пакета
sudo cp /etc/turnserver.conf /etc/turnserver.conf.orig
sudo tee /etc/turnserver.conf > /dev/null <<'EOF'
# /etc/turnserver.conf - TURN-сервер для звонков через XMPP

# Порт STUN/TURN (UDP и TCP)
listening-port=3478

# Диапазон UDP-портов для передачи медиа. Должен совпадать с правилом файрвола.
min-port=50000
max-port=50100

# Только если хостер использует NAT (публичный IP не назначен на интерфейс VPS):
#external-ip=<IP_VPS>/<ВНУТРЕННИЙ_IP>

# Временные учетные данные выдает Prosody (mod_turn_external).
# Секрет совпадает с turn_external_secret в /etc/prosody/prosody.cfg.lua.
use-auth-secret
static-auth-secret=<TURN_SECRET>
realm=turn.inkov.dev

# TLS не используется (см. раздел 8.4)
no-tls
no-dtls

# Безопасность
fingerprint
no-cli
no-multicast-peers
no-software-attribute

# Запрет пересылки на локальные, частные и служебные адреса.
# Без этого через TURN можно обратиться к сервисам внутри VPS, включая панель 3x-ui.
denied-peer-ip=0.0.0.0-0.255.255.255
denied-peer-ip=10.0.0.0-10.255.255.255
denied-peer-ip=100.64.0.0-100.127.255.255
denied-peer-ip=127.0.0.0-127.255.255.255
denied-peer-ip=169.254.0.0-169.254.255.255
denied-peer-ip=172.16.0.0-172.31.255.255
denied-peer-ip=192.0.0.0-192.0.0.255
denied-peer-ip=192.0.2.0-192.0.2.255
denied-peer-ip=192.88.99.0-192.88.99.255
denied-peer-ip=192.168.0.0-192.168.255.255
denied-peer-ip=198.18.0.0-198.19.255.255
denied-peer-ip=198.51.100.0-198.51.100.255
denied-peer-ip=203.0.113.0-203.0.113.255
denied-peer-ip=224.0.0.0-255.255.255.255
denied-peer-ip=::1
denied-peer-ip=::ffff:0.0.0.0-::ffff:255.255.255.255
denied-peer-ip=64:ff9b::-64:ff9b::ffff:ffff
denied-peer-ip=100::-100::ffff:ffff:ffff:ffff
denied-peer-ip=2001::-2001:1ff:ffff:ffff:ffff:ffff:ffff:ffff
denied-peer-ip=2002::-2002:ffff:ffff:ffff:ffff:ffff:ffff:ffff
denied-peer-ip=fc00::-fdff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
denied-peer-ip=fe80::-febf:ffff:ffff:ffff:ffff:ffff:ffff:ffff

# Журнал в systemd journal: journalctl -u coturn
syslog
EOF
# Секрет TURN из файла вместо <TURN_SECRET>
sudo sh -c 'sed -i "s/<TURN_SECRET>/$(cat /root/.secrets/turn_secret)/" /etc/turnserver.conf'
# Файл читают только root и группа turnserver
sudo chown root:turnserver /etc/turnserver.conf
sudo chmod 640 /etc/turnserver.conf
```

Если в разделе 2.5 выяснилось, что VPS работает за NAT, раскомментируйте строку `external-ip` и укажите публичный и внутренний адреса: `sudo nano /etc/turnserver.conf`.

Назначение параметров:

- `use-auth-secret` и `static-auth-secret` - Prosody выдает клиентам временные логин и пароль, вычисленные из общего секрета. Постоянные пароли TURN не нужны.
- `min-port` и `max-port` - 101 порт для медиа. Один звонок через TURN занимает несколько портов, для личного сервера этого достаточно.
- `no-cli` отключает консоль управления coturn, `no-software-attribute` скрывает версию coturn в ответах.
- `denied-peer-ip` - запрет пересылки на внутренние адреса. Без него пользователь TURN может обращаться к сервисам, которые слушают только `127.0.0.1` или внутреннюю сеть хостера.

### 8.3. Запуск и проверка

```bash
sudo systemctl start coturn
# Автозапуск при загрузке: ожидается enabled
systemctl is-enabled coturn
systemctl status coturn --no-pager
sudo ss -tulpn | grep turnserver
```

**Результат:** `Active: active (running)`, `turnserver` слушает порт 3478 по UDP и TCP.

Порты TURN в файрволе: STUN/TURN по TCP и UDP и диапазон передачи медиа.

```bash
sudo ufw allow 3478 comment 'STUN/TURN'
sudo ufw allow 50000:50100/udp comment 'TURN relay'
sudo ufw status | grep -E '3478|50000'
```

Если у хостера есть внешний файрвол (раздел 2.4), откройте те же порты в нем.

Проверка связки Prosody и coturn. Команда читает настройки `turn_external_*` из конфигурации Prosody, перезапуск Prosody для нее не нужен. Имя `turn.inkov.dev` должно уже разрешаться (раздел 3).

```bash
# Выдача учетных данных и выделение relay-порта
sudo prosodyctl check turn -v
# То же плюс пересылка пакета внешнему STUN-серверу через relay
sudo prosodyctl check turn -v --ping=stun.conversations.im
```

**Результат:**

```text
Identified 1 TURN services.

Testing TURN service turn.inkov.dev:3478...
External IP: 203.0.113.10
Relayed address 1: 203.0.113.10:50073
Success!

All checks passed, congratulations!
```

Порт в `Relayed address` находится в диапазоне 50000-50100. Предупреждение `STUN returned a private IP! Is the TURN server behind a NAT and misconfigured?` означает, что VPS за NAT и нужен параметр `external-ip`.

### 8.4. TLS для coturn (опционально)

TURN поверх TLS на порту 5349 помогает клиентам в сетях, где закрыт UDP, но разрешены TCP-соединения с TLS. Порт 443 для этого использовать нельзя: он занят Xray.

Скрипт копирования сертификата для coturn. Скрипты из `/etc/letsencrypt/renewal-hooks/deploy` certbot выполняет после каждого продления.

```bash
sudo install -d -o root -g turnserver -m 750 /etc/coturn/certs
sudo install -d /etc/letsencrypt/renewal-hooks/deploy
sudo tee /etc/letsencrypt/renewal-hooks/deploy/coturn.sh > /dev/null <<'EOF'
#!/bin/sh
# Копирует сертификат Let's Encrypt для coturn и перезапускает службу
set -e
install -o turnserver -g turnserver -m 644 /etc/letsencrypt/live/inkov.dev/fullchain.pem /etc/coturn/certs/fullchain.pem
install -o turnserver -g turnserver -m 600 /etc/letsencrypt/live/inkov.dev/privkey.pem /etc/coturn/certs/privkey.pem
systemctl restart coturn
EOF
sudo chmod 755 /etc/letsencrypt/renewal-hooks/deploy/coturn.sh
```

Включение TLS в coturn и Prosody:

```bash
# coturn: убрать запрет TLS и добавить порт и сертификат
sudo sed -i -e '/^no-tls$/d' -e '/^no-dtls$/d' /etc/turnserver.conf
sudo tee -a /etc/turnserver.conf > /dev/null <<'EOF'

# TLS (раздел 8.4)
tls-listening-port=5349
cert=/etc/coturn/certs/fullchain.pem
pkey=/etc/coturn/certs/privkey.pem
no-tlsv1
no-tlsv1_1
EOF
# Prosody: сообщать клиентам порт TURN с TLS
sudo sed -i 's/^turn_external_tcp = true$/&\nturn_external_tls_port = 5349/' /etc/prosody/prosody.cfg.lua
# Файрвол: TCP для TLS и UDP для DTLS
sudo ufw allow 5349 comment 'TURN TLS'
# Первое копирование сертификата и перезапуск coturn
sudo /etc/letsencrypt/renewal-hooks/deploy/coturn.sh
sudo systemctl reload prosody
```

Проверка:

```bash
openssl s_client -connect turn.inkov.dev:5349 </dev/null 2>/dev/null | openssl x509 -noout -enddate -text | grep -E 'notAfter=|DNS:'
```

**Результат:** строка с датой окончания и строка с `DNS:inkov.dev` и `DNS:*.inkov.dev`. При каждом продлении сертификата (примерно раз в 60 дней) coturn перезапускается, текущие звонки через TURN в этот момент прерываются.

---

## 9. Запуск и проверка

### 9.1. Перезапуск Prosody

```bash
sudo systemctl restart prosody
systemctl status prosody --no-pager
# Последние записи журнала и файл ошибок
sudo tail -n 30 /var/log/prosody/prosody.log
sudo cat /var/log/prosody/prosody.err
```

**Результат:** `Active: active (running)`. В журнале есть строки:

```text
portmanager	info	Activated service 'c2s' on [*]:5222, [::]:5222
portmanager	info	Activated service 's2s' on [*]:5269, [::]:5269
portmanager	info	Activated service 'http' on [127.0.0.1]:5280
portmanager	info	Activated service 'https' on [*]:5281, [::]:5281
upload.inkov.dev:http	info	Serving 'file_share' at https://upload.inkov.dev:5281/file_share
inkov.dev:tls	info	Certificates loaded
```

`prosody.err` пустой или отсутствует. Если служба не запустилась, причина видна в `journalctl -u prosody -n 50 --no-pager`.

### 9.2. Импорт сертификата

Эту же команду certbot выполняет после каждого продления. Теперь в конфигурации есть хост `inkov.dev` и компоненты, поэтому ручной запуск тоже работает:

```bash
sudo prosodyctl --root cert import /etc/letsencrypt/live
```

**Результат:** `Imported certificate and key for hosts inkov.dev, upload.inkov.dev, conference.inkov.dev`. Prosody перечитывает сертификаты автоматически, в журнале появляется `Certificates reloaded`.

### 9.3. Встроенные проверки Prosody

```bash
sudo prosodyctl check config
sudo prosodyctl check dns
sudo prosodyctl check certs
sudo prosodyctl check turn -v
sudo prosodyctl check connectivity
sudo prosodyctl check features
```

| Команда | Что проверяет | Нормальный результат |
|---|---|---|
| `check config` | синтаксис и параметры конфигурации | `All checks passed, congratulations!` |
| `check dns` | SRV-записи, A и AAAA-записи сервера и компонентов, совпадение адресов с IP VPS | `All checks passed, congratulations!` |
| `check certs` | наличие сертификатов для `inkov.dev` и компонентов, имена и срок действия | для каждого хоста `Certificate: /etc/prosody/certs/inkov.dev.crt` |
| `check turn -v` | выдачу учетных данных TURN и выделение relay-порта | `Success!` |
| `check connectivity` | доступность портов 5222 и 5269 из интернета через сервис observe.jabber.network | `xmpp-client: Works`, `xmpp-server: Works` |
| `check features` | набор функций для клиентов | все пункты `OK`, кроме `Web connections` |

Особенности этой схемы:

- `check dns` проверяет цели SRV-записей. A-записи `inkov.dev` с адресами GitHub Pages при наличии SRV-записей не проверяются. Сообщение `inkov.dev A record points to unknown address 185.199.x.x` означает, что SRV-записи не видны серверу: вернитесь к разделу 3.3.
- `check features` отмечает `(!) Web connections`: это BOSH и WebSocket для браузерных клиентов. Мобильным и настольным клиентам они не нужны.
- `check connectivity` обращается к внешнему сервису. Если он недоступен из сети хостера, сообщение `Failed to request check at API` не означает проблем с сервером.
- Если VPS работает за NAT, `check dns` сообщает, что адреса из DNS не найдены на сервере. Укажите публичный адрес в глобальной части конфигурации, выше строки `VirtualHost`: `external_addresses = { "<IP_VPS>" }`.

### 9.4. Проверка снаружи

Где выполняется: локальный компьютер с Linux или macOS. В Windows порты проверяются в PowerShell: `Test-NetConnection xmpp.inkov.dev -Port 5222`, ожидается `TcpTestSucceeded : True`.

```bash
# SRV-записи через обычный резолвер
dig +short SRV _xmpp-client._tcp.inkov.dev
dig +short SRV _xmpp-server._tcp.inkov.dev

# Доступность портов
nc -vz xmpp.inkov.dev 5222
nc -vz xmpp.inkov.dev 5269
nc -vz upload.inkov.dev 5281

# Сертификат на порту клиентов: должен содержать inkov.dev
openssl s_client -connect xmpp.inkov.dev:5222 -starttls xmpp -xmpphost inkov.dev </dev/null 2>/dev/null \
  | openssl x509 -noout -enddate -text | grep -E 'notAfter=|DNS:'
# Сертификат на порту серверов
openssl s_client -connect xmpp.inkov.dev:5269 -starttls xmpp-server -xmpphost inkov.dev </dev/null 2>/dev/null \
  | openssl x509 -noout -enddate -text | grep -E 'notAfter=|DNS:'
# Сертификат HTTPS для обмена файлами
openssl s_client -connect upload.inkov.dev:5281 -servername upload.inkov.dev </dev/null 2>/dev/null \
  | openssl x509 -noout -enddate -text | grep -E 'notAfter=|DNS:'
```

**Результат:** SRV-записи как в разделе 3.3; `nc` сообщает об успешном подключении (`Connected to` или `succeeded`); для каждого порта выводится дата окончания сертификата и строка с `DNS:inkov.dev` и `DNS:*.inkov.dev`.

### 9.5. Внешние сервисы проверки

- https://compliance.conversations.im проверяет поддержку расширений XMPP, важных для мобильных клиентов. Сервис входит на сервер под учетной записью, поэтому используйте временную `test@inkov.dev` (раздел 10.1) и удалите ее после проверки.
- Доступность внешних сервисов из России и сети хостера может меняться. Основная проверка - команды из 9.3 и 9.4.

---

## 10. Пользователи и клиенты

### 10.1. Учетные записи

```bash
# Администратор (адрес указан в admins). Пароль вводится дважды и не попадает в историю shell
sudo prosodyctl adduser admin@inkov.dev
# Тестовый пользователь
sudo prosodyctl adduser test@inkov.dev
# Список пользователей
sudo prosodyctl shell user list inkov.dev
# Фактическая роль администратора
sudo prosodyctl shell user role admin@inkov.dev
```

**Результат:** `OK: Created admin@inkov.dev with role 'prosody:member'` - роль при создании. Права администратора дает список `admins`, поэтому `user role` показывает `OK: prosody:operator`.

Другие операции:

```bash
# Смена пароля
sudo prosodyctl passwd test@inkov.dev
# Удаление учетной записи
sudo prosodyctl deluser test@inkov.dev
```

Приглашение (опционально). Человек сам задает пароль, администратору не нужно его знать:

```bash
sudo prosodyctl shell invite create_account friend@inkov.dev
```

**Результат:** `OK: xmpp:friend@inkov.dev?register;preauth=...`. Ссылку открывают Conversations и основанные на нем клиенты (Monocles Chat, Cheogram): они создают учетную запись с паролем, который задает пользователь. Срок действия приглашения задается флагом `--expires-after`.

### 10.2. Клиенты

| Платформа | Клиент | Источник |
|---|---|---|
| Android | Conversations | F-Droid: https://f-droid.org/packages/eu.siacs.conversations/, сайт https://conversations.im |
| Android | Monocles Chat | F-Droid: https://f-droid.org/packages/de.monocles.chat/ |
| Android | Cheogram | https://cheogram.com |
| iOS, macOS | Monal | App Store, сайт https://monal-im.org |
| Linux | Dino | https://dino.im, пакеты дистрибутива или Flathub |
| Linux, Windows | Gajim | https://gajim.org |

Conversations в Google Play платный, а оплата покупок в Google Play из России недоступна с 2022 года. Версия в F-Droid бесплатная.

### 10.3. Подключение клиента

- Адрес: `user@inkov.dev`, пароль из `prosodyctl adduser`.
- Сервер и порт указывать не нужно: клиент находит `xmpp.inkov.dev:5222` по SRV-записи.
- Если клиент не находит сервер (например, DNS в сети не отвечает на SRV-запросы), укажите вручную в расширенных настройках учетной записи: сервер `xmpp.inkov.dev`, порт `5222`.

### 10.4. Проверка функций

1. Войдите как `admin@inkov.dev` на телефоне и как `test@inkov.dev` на компьютере.
2. Добавьте друг друга в контакты и обменяйтесь сообщениями.
3. Проверьте шифрование OMEMO: в Conversations оно включено по умолчанию, в Gajim и Dino включается в окне чата.
4. Отправьте фотографию: файл загружается на `https://upload.inkov.dev:5281/...`.
5. Создайте групповой чат: его адрес будет вида `room@conference.inkov.dev`.
6. Позвоните между устройствами в разных сетях, например через мобильный интернет и Wi-Fi: так проверяется TURN.
7. Добавьте контакт с другого XMPP-сервера и обменяйтесь сообщениями: так проверяется федерация.
8. Удалите тестового пользователя: `sudo prosodyctl deluser test@inkov.dev`.

---

## 11. Безопасность и обслуживание

### 11.1. Доступ

- Регистрация закрыта (`allow_registration = false`), учетные записи создает только администратор.
- Для XMPP, 3x-ui и SSH используйте разные длинные пароли.
- Порт 5280 и консоль `prosodyctl shell` доступны только локально, наружу их не открывайте.
- Токен Cloudflare дает право менять DNS зоны `inkov.dev`. При подозрении на утечку удалите его в **My Profile → API Tokens** и создайте новый, затем обновите `/root/.secrets/cloudflare.ini`.

### 11.2. Обновления

- Обновления безопасности Ubuntu устанавливает служба `unattended-upgrades`, она включена по умолчанию: `systemctl status unattended-upgrades --no-pager`.
- Пакеты из репозитория Prosody `unattended-upgrades` по умолчанию не обновляет. Раз в месяц выполняйте:

```bash
sudo apt update
apt list --upgradable
sudo apt upgrade
```

- Пакет `prosody` следует за актуальной стабильной версией, включая переход на следующую основную версию. Перед таким обновлением прочитайте заметки к выпуску на https://prosody.im. Чтобы обновлять Prosody только вручную: `sudo apt-mark hold prosody`, снять блокировку: `sudo apt-mark unhold prosody`.

### 11.3. Резервное копирование

```bash
# Конфигурация, данные Prosody, сертификаты и секреты. Загруженные пользователями файлы не включаются
sudo tar --exclude='*/http_file_share' -czf /root/xmpp-backup-$(date +%F).tar.gz \
  /etc/prosody /var/lib/prosody /etc/letsencrypt /etc/turnserver.conf /root/.secrets
sudo ls -lh /root/xmpp-backup-*.tar.gz
```

Сообщение `tar: Removing leading '/' from member names` ожидаемо. Копирование архива на локальный компьютер:

```bash
# На VPS: копия архива в домашний каталог пользователя с правами только для него
sudo install -m 600 -o "$USER" /root/xmpp-backup-$(date +%F).tar.gz ~/
# На локальном компьютере
scp <user>@xmpp.inkov.dev:~/xmpp-backup-*.tar.gz .
# На VPS: удаление копии из домашнего каталога
rm ~/xmpp-backup-*.tar.gz
```

Восстановление: `sudo tar -xzf xmpp-backup-<дата>.tar.gz -C /` и `sudo systemctl restart prosody coturn`.

> **Важно:** архив содержит закрытый ключ сертификата, токен Cloudflare и секрет TURN. Храните копию в зашифрованном виде.

### 11.4. Журналы

- Prosody: `/var/log/prosody/prosody.log` и `prosody.err`. Пакет настраивает ротацию: ежедневно, 14 файлов.
- coturn: `journalctl -u coturn`.
- certbot: `/var/log/letsencrypt/letsencrypt.log`.

### 11.5. fail2ban

Опционально. Prosody 13 по умолчанию не записывает в журнал IP-адреса неудачных попыток входа, поэтому для fail2ban нужен сторонний модуль `mod_log_auth` (https://modules.prosody.im/mod_log_auth). При длинных случайных паролях подбор перебором неэффективен, поэтому для личного сервера fail2ban необязателен.

### 11.6. Переход на Ubuntu 24.04

Стандартная поддержка Ubuntu 22.04 заканчивается в 2027 году. При переходе:

1. Сделайте резервную копию (11.3).
2. `do-release-upgrade` отключает сторонние репозитории. После обновления подключите репозиторий Prosody для 24.04:

```bash
sudo wget https://prosody.im/downloads/repos/noble/prosody.sources -O /etc/apt/sources.list.d/prosody.sources
sudo apt update && sudo apt upgrade
```

3. Проверьте службы: `sudo prosodyctl check`, `systemctl status coturn --no-pager`, `sudo certbot renew --dry-run`.

---

## 12. Устранение неполадок

| Симптом | Вероятная причина | Как проверить | Как исправить |
|---|---|---|---|
| Prosody не запускается, в журнале `Prosody is no longer compatible with Lua 5.1` | Установлен `luarocks` или `lua5.1`, команда `lua` указывает на Lua 5.1 | `readlink -f /usr/bin/lua` | `sudo update-alternatives --set lua-interpreter /usr/bin/lua5.4`, затем `sudo systemctl restart prosody` |
| Prosody не запускается после правки конфигурации | Синтаксическая ошибка Lua: пропущена кавычка, скобка или `;` | `sudo prosodyctl check config` показывает строку с ошибкой | Исправить строку или вернуть `/etc/prosody/prosody.cfg.lua.orig` |
| В журнале `Address already in use` | Порт занят другим процессом | `sudo ss -tulpn`, найти порт в выводе | Освободить порт или сменить его в конфигурации и SRV-записях |
| Клиент не находит сервер | SRV-записи не созданы или еще не видны резолверу; закрыт порт 5222 | `dig +short SRV _xmpp-client._tcp.inkov.dev`, `nc -vz xmpp.inkov.dev 5222` | Проверить записи (раздел 3), файрвол VPS и хостера |
| Клиент сообщает об ошибке сертификата | В сертификате нет `inkov.dev`, сертификат не импортирован или истек | Команды `openssl` из 9.4, `sudo prosodyctl check certs` | Выпустить сертификат с `-d inkov.dev -d '*.inkov.dev'`, выполнить `sudo prosodyctl --root cert import /etc/letsencrypt/live` |
| certbot: `Error determining zone_id: 9109 ... Did you enter a valid Cloudflare Token?` | Неверный токен | Проверка токена из 6.3 | Скопировать токен заново или создать новый |
| certbot: `Unable to determine zone_id for inkov.dev` | У токена нет доступа к зоне | Настройки токена в Cloudflare | Zone Resources: `Include` / `Specific zone` / `inkov.dev` |
| certbot: `Error communicating with the Cloudflare API` с подсказкой про `Zone:DNS:Edit` | У токена нет права редактировать DNS | Permissions токена | `Zone` / `DNS` / `Edit` |
| certbot или проверка токена завершается по таймауту | API Cloudflare недоступен из сети хостера | Проверка токена из 6.3 | Ручная проверка DNS-01 (6.8, вариант 1) |
| certbot: `DNS problem: NXDOMAIN looking up TXT` или `Incorrect TXT record` | TXT-запись не успела распространиться | Повторить команду | Увеличить `--dns-cloudflare-propagation-seconds` до 120 |
| Сообщения на другие серверы не доходят | Закрыт порт 5269, нет SRV `_xmpp-server`, у удаленного сервера недействительный сертификат | `nc -vz xmpp.inkov.dev 5269`, `sudo tail -n 50 /var/log/prosody/prosody.log` | Открыть порт, проверить SRV; для отдельного домена с недействительным сертификатом добавить `s2s_insecure_domains = { "example.org" }` в глобальную часть конфигурации |
| Не отправляются файлы | Закрыт порт 5281, нет записи `upload`, превышен лимит | `nc -vz upload.inkov.dev 5281`, проверка сертификата на 5281 из 9.4 | Открыть порт, создать запись, изменить лимиты в компоненте `upload.inkov.dev` |
| Звонки не соединяются или нет звука | coturn остановлен, закрыт диапазон 50000-50100/udp, секреты не совпадают, VPS за NAT без `external-ip` | `systemctl status coturn`, `sudo prosodyctl check turn -v --ping=stun.conversations.im` | Запустить coturn, открыть порты, повторить подстановку секрета, указать `external-ip` |
| `check turn`: `STUN returned a private IP` | VPS за NAT | `ip -br addr` | `external-ip=<IP_VPS>/<ВНУТРЕННИЙ_IP>` в `/etc/turnserver.conf`, затем `sudo systemctl restart coturn` |
| Prosody: `Permission denied` при загрузке ключа | В конфигурации указан путь к ключу в `/etc/letsencrypt` | `sudo cat /var/log/prosody/prosody.err` | Удалить явные пути к сертификатам, выполнить `cert import` |
| После включения ufw пропал доступ по SSH или перестал работать VPN | Не разрешены порты SSH или 3x-ui | Консоль хостера: `sudo ufw status numbered` | `sudo ufw allow <PORT>/tcp` или `sudo ufw disable` |
| Подключение работает не во всех сетях | В сети клиента закрыты нестандартные порты | Проверить из другой сети | Подключение и обмен файлами через порт 443 (приложение А) |
| `check dns`: `inkov.dev A record points to unknown address 185.199.x.x` | SRV-записи не видны резолверу сервера | `dig +short SRV _xmpp-client._tcp.inkov.dev` на VPS | Проверить SRV-записи в Cloudflare, подождать до 30 минут |

Подробный журнал Prosody: в `/etc/prosody/prosody.cfg.lua` замените `info =` на `debug =` в блоке `log`, выполните `sudo systemctl restart prosody`, воспроизведите проблему и верните `info`.

---

## 13. Резюме

1. Prosody 13 на VPS обслуживает адреса `user@inkov.dev`, сайт `inkov.dev` остается на GitHub Pages.
2. SRV-записи в зоне `inkov.dev` направляют клиентов и другие серверы на `xmpp.inkov.dev`.
3. Сертификат на `inkov.dev` и `*.inkov.dev` выпускается через DNS-01 и API Cloudflare, продлевается автоматически и импортируется в Prosody.
4. Обмен файлами работает на порту 5281, звонки - через coturn. Порт 443 остается за Xray, конфигурация 3x-ui не менялась.
5. Регистрация закрыта, шифрование обязательно, coturn не пересылает трафик на внутренние адреса.
6. Опционально: подключение клиентов и обмен файлами через порт 443 рядом с REALITY (приложение А).

### Чек-лист

- [ ] Порты Prosody и coturn свободны, база 3x-ui скопирована (раздел 2)
- [ ] DNS-записи созданы в режиме DNS only и видны через `dig` (раздел 3)
- [ ] Prosody 13 установлен с `--no-install-recommends`, `prosodyctl about` показывает Lua 5.4 (раздел 4)
- [ ] Правила ufw добавлены, SSH и VPN работают (раздел 5)
- [ ] Сертификат выпущен, `sudo certbot renew --dry-run` проходит (раздел 6)
- [ ] `sudo prosodyctl check config` без ошибок (раздел 7)
- [ ] `sudo prosodyctl check turn -v` показывает `Success!` (раздел 8)
- [ ] `check dns`, `check certs`, `check connectivity` без ошибок (раздел 9)
- [ ] Сообщения, файлы, групповые чаты, звонки и федерация работают (раздел 10)
- [ ] Резервная копия создана и перенесена с сервера (раздел 11)

### Меры безопасности

Сервер доступен из интернета, поэтому его безопасность зависит от регулярного обслуживания. Раз в месяц обновляйте пакеты и проверяйте срок сертификата (`sudo certbot certificates`). Не открывайте наружу порт 5280 и не отключайте `s2s_secure_auth` и `c2s_require_encryption`. Храните файлы из `/root/.secrets` и резервные копии в защищенном месте. После любых изменений файрвола проверяйте доступ по SSH и работу VPN.

### Документация

- Prosody: https://prosody.im/doc
- Репозиторий пакетов Prosody: https://prosody.im/download/package_repository
- Конфигурация: https://prosody.im/doc/configure
- Сертификаты: https://prosody.im/doc/certificates, Let's Encrypt: https://prosody.im/doc/letsencrypt
- DNS: https://prosody.im/doc/dns
- TURN: https://prosody.im/doc/turn
- mod_http_file_share: https://prosody.im/doc/modules/mod_http_file_share
- mod_turn_external: https://prosody.im/doc/modules/mod_turn_external
- mod_mam: https://prosody.im/doc/modules/mod_mam
- mod_muc: https://prosody.im/doc/modules/mod_muc
- prosodyctl: https://prosody.im/doc/prosodyctl
- Зависимости и версии Lua: https://prosody.im/doc/depends
- coturn: https://github.com/coturn/coturn
- certbot: https://eff-certbot.readthedocs.io/en/stable/
- Плагин certbot для Cloudflare: https://certbot-dns-cloudflare.readthedocs.io/en/stable/
- DNS в Cloudflare: https://developers.cloudflare.com/dns/
- API-токены Cloudflare: https://developers.cloudflare.com/fundamentals/api/get-started/create-token/
- REALITY в Xray: https://xtls.github.io/en/config/transports/reality.html
- HAProxy 2.4: https://docs.haproxy.org/2.4/configuration.html

---

## Приложение А. XMPP и обмен файлами через порт 443

Необязательное дополнение для случая, когда клиенты подключаются из сетей, где разрешен только порт 443. Приложение меняет одну настройку 3x-ui, поэтому выполняйте его, когда основная схема из разделов 1-10 уже работает.

### А.1. Как это работает

Xray с REALITY на порту 443 передает в адрес из поля Target все соединения, которые не прошли проверку REALITY. Обычно в Target указан внешний сайт для маскировки. В этой схеме Target меняется на локальный HAProxy. HAProxy читает имя сервера (SNI) из открытой части TLS-рукопожатия, трафик не расшифровывает и направляет соединение по имени:

```text
Клиент ──443──> Xray (REALITY)
                ├── клиент VPN (проверка REALITY пройдена) ──> VPN, как раньше
                └── остальные соединения ──> Target: HAProxy 127.0.0.1:8443
                                              ├── SNI inkov.dev        ──> Prosody 127.0.0.1:5223, прямой TLS
                                              ├── SNI upload.inkov.dev ──> Prosody 127.0.0.1:5281, HTTPS
                                              └── остальные SNI        ──> прежний Target (сайт маскировки)
```

TLS завершает сам Prosody со своим сертификатом Let's Encrypt, Xray и HAProxy передают байты без изменений. Через порт 443 начинают работать:

- подключение клиентов по прямому TLS (XEP-0368): новая SRV-запись `_xmpps-client._tcp` указывает порт 443;
- обмен файлами: ссылки на файлы вида `https://upload.inkov.dev/file_share/...` без номера порта.

Связь с другими серверами (5269) и звонки (3478 и 50000-50100/udp) остаются на прежних портах.

### А.2. Условия и риски

- **Схема подходит только для inbound с Security = reality.** Если у inbound на порту 443 в 3x-ui Security = tls, Xray сам расшифровывает соединения и передает в fallback уже открытый трафик. Для клиентов XMPP такой вариант не работает: Prosody получает соединение без TLS и при `c2s_require_encryption = true` не принимает вход.
- **VPN начинает зависеть от HAProxy.** REALITY обращается к Target при каждом подключении, в том числе при подключении клиентов VPN. На тестовом стенде при остановленном HAProxy клиенты VPN не подключались.
- **Меняется конфигурация 3x-ui:** одно поле Target. Это отступление от принципа из раздела 1.3.
- Для Prosody клиенты, подключенные через 443, приходят с адреса 127.0.0.1. Реальных IP-адресов этих клиентов в журнале Prosody нет.
- Трафик не становится скрытым: имена `inkov.dev`, `upload.inkov.dev` и признак протокола XMPP (ALPN `xmpp-client`) передаются в открытой части TLS-рукопожатия. Меняется только порт.
- Если в настройках REALITY заданы ограничения скорости для соединений, не прошедших проверку (`limitFallbackUpload`, `limitFallbackDownload`, по умолчанию выключены), они действуют и на XMPP через 443.

Схема проверена на стенде: Xray 26.3.27 с REALITY, HAProxy 2.4 из Ubuntu 22.04, Prosody 13.0.7. Через порт 443 прошли вход XMPP-клиента, выдача слота и загрузка файла, подключение клиента VPN, ответ сайта маскировки на чужой SNI и на соединение без SNI, ClientHello размером около 1,5 КБ и ClientHello, пришедший двумя частями.

### А.3. Текущие настройки REALITY

Где выполняется: панель 3x-ui.

1. Откройте **Inbounds** (**Входящие**), найдите inbound на порту 443 и откройте его редактирование.
2. Убедитесь, что Security (Безопасность) = `reality`.
3. Запишите значения полей **Target** (**Цель**, в старых версиях панели - **Dest**) и **SNI**. Target - адрес с портом, например `www.microsoft.com:443`. Далее это значение обозначается `<REALITY_TARGET>`, а первое имя из SNI - `<REALITY_SNI>`.
4. Проверьте, что **Xver** = `0`. При другом значении Xray добавляет к соединению заголовок PROXY, и HAProxy в этой настройке не прочитает SNI.

На этом шаге ничего не меняйте.

### А.4. HAProxy

```bash
sudo apt install haproxy
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.orig
sudo tee /etc/haproxy/haproxy.cfg > /dev/null <<'EOF'
# /etc/haproxy/haproxy.cfg - маршрутизация по SNI за REALITY (приложение А)

global
    log /dev/log local0
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    # Для установленных соединений: сессии XMPP длятся часами
    timeout tunnel 1h

# Сюда REALITY передает соединения, которые не прошли его проверку
frontend reality_target
    bind 127.0.0.1:8443
    # Ожидание TLS ClientHello, чтобы прочитать SNI. Трафик не расшифровывается
    tcp-request inspect-delay 5s
    tcp-request content accept if { req.ssl_hello_type 1 }
    use_backend prosody_c2s if { req.ssl_sni -i inkov.dev }
    use_backend prosody_upload if { req.ssl_sni -i upload.inkov.dev }
    default_backend reality_original_target

# Клиенты XMPP с прямым TLS (XEP-0368)
backend prosody_c2s
    server prosody 127.0.0.1:5223

# Обмен файлами
backend prosody_upload
    server prosody 127.0.0.1:5281

# Прежний Target из настроек REALITY: сайт для маскировки
backend reality_original_target
    server target <REALITY_TARGET>
EOF
# Прежнее значение Target из 3x-ui вместо плейсхолдера. Пример значения замените своим
sudo sed -i 's|<REALITY_TARGET>|www.microsoft.com:443|' /etc/haproxy/haproxy.cfg
# Проверка синтаксиса и запуск
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
systemctl is-enabled haproxy
```

**Результат:** `Configuration file is valid` и `enabled`.

Если прежний Target указывает на локальный порт 8443 (например, `127.0.0.1:8443` или просто `8443`), выберите для HAProxy другой свободный порт и укажите его в строке `bind` и в А.6.

Проверка маршрута к прежнему Target. VPN на этом шаге работает по-старому:

```bash
openssl s_client -connect 127.0.0.1:8443 -servername <REALITY_SNI> </dev/null 2>/dev/null | openssl x509 -noout -subject
```

**Результат:** `subject` сертификата сайта из Target. Пустой вывод означает, что HAProxy не подключился к Target: проверьте адрес в `/etc/haproxy/haproxy.cfg` и журнал `sudo tail -n 20 /var/log/haproxy.log`.

### А.5. Prosody

```bash
# Резервная копия перед изменениями
sudo cp /etc/prosody/prosody.cfg.lua /etc/prosody/prosody.cfg.lua.before-443
# Глобальная часть: строки добавляются после https_ports
sudo sed -i '/^https_ports = { 5281 }$/r /dev/stdin' /etc/prosody/prosody.cfg.lua <<'EOF'

-- Прямой TLS для клиентов (XEP-0368). Соединения приходят через порт 443
-- от HAProxy (приложение А), поэтому порт слушается только на 127.0.0.1
c2s_direct_tls_ports = { 5223 }
c2s_direct_tls_interfaces = { "127.0.0.1" }
-- HAProxy передает TLS без изменений и не добавляет X-Forwarded-For,
-- поэтому заголовку от 127.0.0.1 доверять нельзя
trusted_proxies = { }
EOF
# Компонент обмена файлами: ссылки на файлы без порта 5281
sudo sed -i '/^Component "upload.inkov.dev" "http_file_share"$/r /dev/stdin' /etc/prosody/prosody.cfg.lua <<'EOF'
    http_external_url = "https://upload.inkov.dev/" -- ссылки на файлы через порт 443
EOF
# Проверка и применение
sudo grep -n -E 'c2s_direct_tls|trusted_proxies|http_external_url' /etc/prosody/prosody.cfg.lua
sudo prosodyctl check config
sudo systemctl restart prosody
sudo grep -E "c2s_direct_tls' on|Serving 'file_share'" /var/log/prosody/prosody.log | tail -n 2
```

**Результат:** `grep` находит четыре строки: `c2s_direct_tls_ports`, `c2s_direct_tls_interfaces`, `trusted_proxies`, `http_external_url`. `check config` выводит `All checks passed, congratulations!`. В журнале две строки, их порядок может отличаться:

```text
upload.inkov.dev:http	info	Serving 'file_share' at https://upload.inkov.dev/file_share
portmanager	info	Activated service 'c2s_direct_tls' on [127.0.0.1]:5223
```

Проверка маршрутов HAProxy до Prosody. VPN на этом шаге все еще работает по-старому:

```bash
# Обмен файлами: сертификат Prosody
openssl s_client -connect 127.0.0.1:8443 -servername upload.inkov.dev </dev/null 2>/dev/null \
  | openssl x509 -noout -text | grep DNS:
# Прямой TLS: Prosody отвечает списком механизмов входа
(printf "<?xml version='1.0'?><stream:stream to='inkov.dev' xmlns='jabber:client' xmlns:stream='http://etherx.jabber.org/streams' version='1.0'>"; sleep 2) \
  | timeout 5 openssl s_client -connect 127.0.0.1:8443 -servername inkov.dev -alpn xmpp-client -quiet 2>/dev/null \
  | grep -o '<mechanism>[A-Z0-9-]*</mechanism>'
```

**Результат:** строка с `DNS:inkov.dev` и `DNS:*.inkov.dev`, затем строки вида `<mechanism>SCRAM-SHA-1</mechanism>`.

### А.6. Переключение Target в 3x-ui

> **Важно:** после этого шага VPN работает через HAProxy. Подготовьте клиентское устройство для проверки VPN и держите под рукой записанное прежнее значение Target.

1. В 3x-ui откройте тот же inbound на порту 443.
2. В поле **Target** (**Цель**, **Dest**) укажите `127.0.0.1:8443`. SNI, ключи и Short IDs не меняйте, Xver оставьте `0`.
3. Сохраните inbound. Если панель не применила изменения сама, перезапустите Xray из панели.
4. Сразу проверьте подключение VPN с клиентского устройства.

Если VPN не подключается, верните в Target прежнее значение и проверьте HAProxy: `systemctl status haproxy --no-pager` и проверка из А.4.

### А.7. DNS

Где выполняется: панель Cloudflare, **DNS → Records**.

| Действие | Type | Name | Priority | Weight | Port | Target |
|---|---|---|---|---|---|---|
| Добавить | SRV | `_xmpps-client._tcp` | `0` | `5` | `443` | `xmpp.inkov.dev` |
| Изменить | SRV | `_xmpp-client._tcp` | `10` вместо `0` | `5` | `5222` | `xmpp.inkov.dev` |

Клиенты с поддержкой XEP-0368 объединяют записи `_xmpps-client` и `_xmpp-client` и подключаются в порядке приоритета: сначала к порту 443, при ошибке - к 5222. Клиенты без поддержки XEP-0368 продолжают подключаться к 5222.

```bash
NS=$(dig +short NS inkov.dev | head -1)
dig +short @"$NS" SRV _xmpps-client._tcp.inkov.dev
dig +short @"$NS" SRV _xmpp-client._tcp.inkov.dev
```

**Результат:** `0 5 443 xmpp.inkov.dev.` и `10 5 5222 xmpp.inkov.dev.`

### А.8. Проверка

Где выполняется: локальный компьютер.

```bash
# XMPP через порт 443
(printf "<?xml version='1.0'?><stream:stream to='inkov.dev' xmlns='jabber:client' xmlns:stream='http://etherx.jabber.org/streams' version='1.0'>"; sleep 3) \
  | timeout 6 openssl s_client -connect xmpp.inkov.dev:443 -servername inkov.dev -alpn xmpp-client -quiet 2>/dev/null \
  | grep -o '<mechanism>[A-Z0-9-]*</mechanism>'
# Обмен файлами через порт 443
openssl s_client -connect xmpp.inkov.dev:443 -servername upload.inkov.dev </dev/null 2>/dev/null \
  | openssl x509 -noout -enddate -text | grep -E 'notAfter=|DNS:'
# Для чужого SNI по-прежнему отвечает сайт маскировки
openssl s_client -connect xmpp.inkov.dev:443 -servername <REALITY_SNI> </dev/null 2>/dev/null \
  | openssl x509 -noout -subject
```

**Результат:** строки с механизмами входа; дата окончания и `DNS:inkov.dev`, `DNS:*.inkov.dev`; для `<REALITY_SNI>` - сертификат сайта из прежнего Target, как до изменений.

На VPS:

```bash
sudo prosodyctl check dns
sudo prosodyctl check connectivity
```

- `check dns` сообщает `SRV target xmpp.inkov.dev contains unknown Direct TLS client port: 443`. Это ожидаемо: Prosody слушает порт 5223, а порт 443 обслуживают Xray и HAProxy.
- `check connectivity` дополнительно проверяет прямой TLS и выводит `xmpps-client: Works`.

В клиенте переподключите учетную запись и отправьте файл: ссылка начинается с `https://upload.inkov.dev/file_share/`, без `:5281`. Ранее отправленные ссылки с портом 5281 продолжают работать, потому что порт 5281 остается открытым.

### А.9. Обслуживание

- HAProxy определяет IP-адрес Target при запуске и при перезагрузке конфигурации. Если сайт маскировки сменит адрес, VPN перестанет подключаться. Еженедельная перезагрузка конфигурации, при которой установленные соединения не разрываются:

```bash
echo '0 5 * * 1 root systemctl reload haproxy' | sudo tee /etc/cron.d/haproxy-reload
```

- Сайт маскировки теперь задается в `backend reality_original_target` в `/etc/haproxy/haproxy.cfg`. При его смене меняйте этот адрес и SNI в 3x-ui, а Target в 3x-ui оставляйте `127.0.0.1:8443`.
- Журнал HAProxy: `/var/log/haproxy.log`.

### А.10. Откат

1. 3x-ui: верните в Target прежнее значение.
2. Cloudflare: удалите запись `_xmpps-client._tcp`, верните записи `_xmpp-client._tcp` приоритет `0`.
3. Prosody: `sudo cp /etc/prosody/prosody.cfg.lua.before-443 /etc/prosody/prosody.cfg.lua && sudo systemctl restart prosody`. Копия сделана в начале А.5: изменения конфигурации, внесенные после нее, нужно повторить.
4. HAProxy: `sudo systemctl disable --now haproxy` и `sudo rm /etc/cron.d/haproxy-reload`.

### А.11. Неполадки

| Симптом | Вероятная причина | Как проверить | Как исправить |
|---|---|---|---|
| VPN перестал подключаться после смены Target | HAProxy остановлен или не подключается к прежнему Target | `systemctl status haproxy --no-pager`, проверка из А.4 | Запустить HAProxy, исправить адрес в `backend reality_original_target` или вернуть Target (А.10) |
| VPN перестал подключаться через несколько дней или недель | Сайт маскировки сменил IP-адрес | Проверка из А.4 | `sudo systemctl reload haproxy`, проверить задание из А.9 |
| Через 443 XMPP не подключается, через 5222 работает | Нет записи `_xmpps-client`, Prosody не слушает 5223 или клиент не поддерживает XEP-0368 | Проверки из А.5 и А.8 | Исправить запись или конфигурацию. Клиенты без XEP-0368 подключаются к 5222 |
| На `upload.inkov.dev:443` отвечает сайт маскировки | Нет маршрута в HAProxy или Xver не равен 0 | Проверка из А.5 | Проверить строку `use_backend prosody_upload` и поле Xver |
| Ссылки на файлы по-прежнему содержат `:5281` | Не добавлен `http_external_url` или Prosody не перезапущен | Строка `Serving 'file_share' at` в журнале Prosody | Повторить А.5 |
