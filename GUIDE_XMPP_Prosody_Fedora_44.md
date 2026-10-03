# Пошаговое руководство: XMPP-сервер Prosody на чистом VPS в России (Fedora Server 44)

Версии: Fedora Server 44, Prosody 13.0.6, драйвер SQLite из пакета lua-dbi 0.7.5, coturn 4.18.0 и certbot 5.8.0 с плагином dns-cloudflare. Все пакеты из репозиториев Fedora. SELinux работает в режиме enforcing. DNS домена обслуживает Cloudflare. Конфигурации Prosody и coturn, импорт сертификата и модули SASL2, Bind 2 и FAST (раздел 14) проверены на этих версиях.

Руководство рассчитано на отдельный сервер без других сервисов. Схема та же, что в [GUIDE_XMPP_Prosody_Ubuntu_26.04.md](GUIDE_XMPP_Prosody_Ubuntu_26.04.md): Prosody занимает порт 443, данные хранятся в SQLite, звонки идут через coturn. Что в Fedora устроено иначе:

- Prosody ставится из репозитория Fedora. Для Fedora проект Prosody рекомендует именно этот пакет, а репозиторий packages.prosody.im предназначен для Debian и Ubuntu. В Fedora есть только Lua 5.4, поэтому конфликта версий Lua, описанного для Ubuntu, здесь нет.
- SELinux работает в режиме enforcing. Чтобы Prosody мог слушать порт 443, включается булев параметр политики `prosody_bind_http_port` (раздел 4.3). Отключать SELinux не нужно.
- Служба `prosody` в Fedora работает от пользователя `prosody` без права слушать порты ниже 1024. Право добавляется файлом drop-in для systemd (раздел 4.3).
- Файрвол - firewalld, а не ufw (раздел 5).
- Сертификаты Prosody хранятся в `/etc/pki/prosody`, конфигурация coturn - в `/etc/coturn/turnserver.conf`, служба coturn работает от пользователя `coturn`.
- Службы не запускаются сразу после установки пакетов. Таймер продления сертификатов `certbot-renew.timer` тоже включается вручную (раздел 6.7).
- Выпуск Fedora поддерживается около 13 месяцев. Fedora 44 получает обновления до июня 2027 года, затем сервер нужно обновить до следующего выпуска (раздел 11.6).
- Раздел 14 (необязательный) описывает SASL2, Bind 2 и FAST: ускоренное переподключение мобильных клиентов.

> **Важно:** руководство проверено на Fedora Server. В облачных образах Fedora (Cloud Edition) может не быть firewalld и Cockpit. Отличия указаны в разделах 2.8 и 5.

## Содержание

1. [Введение](#1-введение)
2. [Подготовка сервера](#2-подготовка-сервера)
3. [DNS-записи в Cloudflare](#3-dns-записи-в-cloudflare)
4. [Установка Prosody](#4-установка-prosody)
5. [Файрвол](#5-файрвол)
6. [SSL-сертификат](#6-ssl-сертификат)
7. [Конфигурация Prosody](#7-конфигурация-prosody)
8. [Звонки: coturn](#8-звонки-coturn)
9. [Запуск и проверка](#9-запуск-и-проверка)
10. [Пользователи и клиенты](#10-пользователи-и-клиенты)
11. [Безопасность и обслуживание](#11-безопасность-и-обслуживание)
12. [Устранение неполадок](#12-устранение-неполадок)
13. [Резюме](#13-резюме)
14. [Быстрое подключение: SASL2, Bind 2, FAST (опционально)](#14-быстрое-подключение-sasl2-bind-2-fast-опционально)

---

## 1. Введение

### 1.1. Что такое XMPP и Prosody

XMPP - открытый протокол обмена сообщениями. Серверы XMPP связаны между собой по тому же принципу, что и почтовые: пользователь сервера `inkov.dev` может переписываться с пользователями любых других XMPP-серверов. Prosody - XMPP-сервер на языке Lua. Он использует десятки мегабайт памяти, настраивается одним файлом и подходит для личного сервера. После выполнения руководства сервер обслуживает адреса вида `user@inkov.dev`, хранит историю сообщений, передает файлы, поддерживает групповые чаты и звонки.

### 1.2. Схема

```text
Cloudflare DNS, зона inkov.dev
├── inkov.dev                     A      GitHub Pages (не меняется)
├── xmpp.inkov.dev                A      <IP_VPS>
├── conference.inkov.dev          CNAME  xmpp.inkov.dev
├── upload.inkov.dev              CNAME  xmpp.inkov.dev
├── turn.inkov.dev                CNAME  xmpp.inkov.dev
├── _xmpps-client._tcp.inkov.dev  SRV    0 5 443 xmpp.inkov.dev
├── _xmpp-client._tcp.inkov.dev   SRV    10 5 5222 xmpp.inkov.dev
└── _xmpp-server._tcp.inkov.dev   SRV    0 5 5269 xmpp.inkov.dev

VPS <IP_VPS>, Fedora Server 44, SELinux enforcing, firewalld
├── SSH            22/tcp
├── Prosody        443/tcp           клиенты (прямой TLS) и HTTPS для обмена файлами
│                  5222/tcp          клиенты (STARTTLS), запасной порт
│                  5269/tcp          связь с другими серверами
│                  5280/tcp          только 127.0.0.1
└── coturn         3478/tcp и udp    STUN/TURN для звонков
                   50000-50100/udp   передача медиа
```

Порядок подключения:

1. Клиент с адресом `user@inkov.dev` запрашивает SRV-записи `_xmpps-client._tcp.inkov.dev` и `_xmpp-client._tcp.inkov.dev`. Клиенты с поддержкой XEP-0368 сначала подключаются к `xmpp.inkov.dev:443` сразу по TLS, при ошибке - к порту 5222 со STARTTLS. Клиенты без поддержки XEP-0368 подключаются к 5222.
2. Prosody предъявляет сертификат на `inkov.dev`.
3. Другие XMPP-серверы находят сервер через SRV-запись `_xmpp-server._tcp.inkov.dev`, порт 5269.
4. Файлы загружаются и скачиваются по адресам вида `https://upload.inkov.dev/file_share/...` через порт 443.
5. Сайт на GitHub Pages продолжает работать: у SRV-записей другие имена, они не пересекаются с A-записями `inkov.dev`.

### 1.3. Принятые решения

- **Адреса `user@inkov.dev`, сервер на `xmpp.inkov.dev`.** В Prosody настроен `VirtualHost "inkov.dev"`, SRV-записи связывают домен с сервером.
- **Сертификат на `inkov.dev` и `*.inkov.dev`.** Клиенты и другие серверы проверяют сертификат по домену из адреса (RFC 6120), а не по имени из SRV-записи. Apex-домен `inkov.dev` указывает на GitHub, поэтому проверку через HTTP (HTTP-01) с VPS пройти нельзя. Используется проверка через DNS (DNS-01): certbot сам создает временную TXT-запись через API Cloudflare. Один wildcard-сертификат покрывает `inkov.dev` и все поддомены.
- **Prosody 13 из репозитория Fedora.** В Fedora 44 версия 13.0.6. Исправления ветки 13.x приходят через репозиторий `updates` вместе с остальными обновлениями системы. Сторонние репозитории не нужны.
- **Порт 443 у Prosody.** Модуль `mod_net_multiplex` принимает на одном порту прямой TLS для клиентов (XEP-0368) и HTTPS для обмена файлами. Сервис выбирается по ALPN, а если клиент ALPN не передал - по первым байтам соединения. Порт 5222 остается для клиентов без поддержки XEP-0368.
- **SELinux в режиме enforcing.** В политике Fedora есть модуль для Prosody: служба получает доступ только к своим файлам и портам. Порт 443 разрешается штатным булевым параметром, собственный модуль политики не нужен.
- **Хранилище SQLite.** Учетные записи, контакты, архив сообщений и данные групповых чатов хранятся в одном файле `/var/lib/prosody/prosody.sqlite`. Копия базы снимается на работающем сервере командой `sqlite3 .backup`, перенос на другой сервер сводится к копированию одного файла. Драйвер входит в пакет `lua-dbi` (раздел 4.2). Загруженные файлы хранятся на диске, а не в базе.
- **Связь с другими серверами только через 5269.** На порту 443 не настроена проверка сертификатов других серверов, а при `s2s_secure_auth = true` без нее серверы не пройдут аутентификацию. Поэтому запись `_xmpps-server` не публикуется.
- **Звонки через coturn.** `mod_turn_external` только выдает клиентам адрес и временные учетные данные TURN-сервера. Сам TURN-сервер ставится отдельно (раздел 8).
- **Сервер отдельный.** Базовая защита сервера входит в руководство (раздел 2).

### 1.4. Ограничения схемы

- Порт 443 занят Prosody. Чтобы позже разместить на этом IP веб-сайт или VPN на порту 443, понадобится маршрутизатор по SNI перед сервисами (например, HAProxy).
- Звонки используют порт 3478 и диапазон 50000-50100/udp. В сетях, где открыт только порт 443, звонки могут не работать.
- Трафик не становится скрытым: имена `inkov.dev`, `upload.inkov.dev` и признак протокола XMPP (ALPN `xmpp-client`) передаются в открытой части TLS-рукопожатия.
- Домен в адресах пользователей меняется только вместе с учетными записями: при смене домена их нужно создать заново.
- Новый выпуск Fedora выходит раз в полгода, каждый выпуск поддерживается около 13 месяцев. Раз в 6-12 месяцев сервер нужно обновлять до следующего выпуска (раздел 11.6). Если нужен редкий цикл обновлений ОС, используйте руководство для Ubuntu 26.04 LTS.

### 1.5. Параметры

| Параметр | Значение | Где используется |
|---|---|---|
| XMPP-домен | `inkov.dev` | адреса пользователей, `VirtualHost` |
| Хост сервера | `xmpp.inkov.dev` | A-запись, цель SRV-записей |
| `<IP_VPS>` | публичный IPv4 VPS, в примерах `203.0.113.10` | DNS, SSH, проверки |
| `<USER>` | имя пользователя с правами sudo | SSH, раздел 2.3 |
| `<EMAIL>` | email для аккаунта Let's Encrypt | certbot |
| `<CF_API_TOKEN>` | API-токен Cloudflare | только файл `/root/.secrets/cloudflare.ini` |
| `<TURN_SECRET>` | общий секрет Prosody и coturn | создается в разделе 7.2, подставляется командой |

Команды в блоках кода выполняются на VPS, если не указано другое. Плейсхолдеры в угловых скобках замените своими значениями.

### 1.6. Что понадобится

- VPS с чистой Fedora Server 44 и доступом по SSH: root или пользователь с sudo, которого создал хостер.
- SSH-ключ на локальном компьютере. Если ключа нет, он создается в разделе 2.3.
- Доступ к панели Cloudflare для домена `inkov.dev`.
- Доступ к консоли VPS в панели хостера (VNC или аналог) на случай ошибки в настройке SSH или файрвола.
- XMPP-клиент на телефоне или компьютере (раздел 10).

---

## 2. Подготовка сервера

### 2.1. ОС и ресурсы

```bash
# Версия ОС: ожидается Fedora release 44
cat /etc/fedora-release
# Редакция: Server Edition или Cloud Edition
grep '^VARIANT=' /etc/os-release
# Режим SELinux: ожидается Enforcing
getenforce
# Свободное место на корневом разделе
df -h /
# Свободная память
free -h
```

**Результат:** `Fedora release 44 (Forty Four)`, `VARIANT="Server Edition"` (в облачных образах `Cloud Edition`), `Enforcing`, свободно не меньше 6 ГБ на диске: до 5 ГБ отводится под файлы пользователей (раздел 7). Если места меньше, уменьшите `http_file_share_global_quota` в конфигурации Prosody. Prosody и coturn вместе используют меньше 100 МБ памяти.

Если `getenforce` выводит `Permissive` или `Disabled`, хостер ослабил или отключил SELinux. Руководство работает и в этом случае, но без защиты, которую дает политика SELinux. Как вернуть режим enforcing, описано в документации Fedora (ссылка в разделе 13).

### 2.2. Зеркала и обновление системы

```bash
sudo dnf makecache
```

**Результат:** список репозиториев и строка `Metadata cache created.` без ошибок `Curl error` и `Failed to download metadata`. В этом случае зеркала менять не нужно: dnf сам выбирает ближайшее зеркало через сервис `mirrors.fedoraproject.org`. Ошибка только для репозитория `fedora-cisco-openh264` (кодеки для рабочих станций) на работу сервера не влияет: dnf пропускает этот репозиторий.

Если dnf не может загрузить метаданные репозиториев `fedora` и `updates`, переключите их на зеркало Яндекса. Переопределение записывается в отдельный файл, файлы пакета `fedora-repos` в `/etc/yum.repos.d` не меняются.

```bash
sudo install -d -m 755 /etc/dnf/repos.override.d
sudo tee /etc/dnf/repos.override.d/80-mirror-yandex.repo > /dev/null <<'EOF'
# Зеркало Яндекса вместо выбора зеркала через mirrors.fedoraproject.org
[fedora]
metalink=
baseurl=https://mirror.yandex.ru/fedora/linux/releases/$releasever/Everything/$basearch/os/

[updates]
metalink=
baseurl=https://mirror.yandex.ru/fedora/linux/updates/$releasever/Everything/$basearch/
EOF
sudo dnf clean all
sudo dnf makecache
# Адреса, которые теперь использует dnf
sudo dnf repo info fedora updates | grep 'Base URL'
```

**Результат:**

```text
  Base URL           : https://mirror.yandex.ru/fedora/linux/releases/44/Everything/x86_64/os/
  Base URL           : https://mirror.yandex.ru/fedora/linux/updates/44/Everything/x86_64/
```

Переменная `$releasever` подставляется автоматически, поэтому файл работает и после перехода на следующий выпуск Fedora. Откат:

```bash
sudo rm /etc/dnf/repos.override.d/80-mirror-yandex.repo && sudo dnf makecache
```

Обновление пакетов:

```bash
sudo dnf upgrade --refresh -y
# Нужна ли перезагрузка после обновления ядра и системных библиотек
sudo dnf needs-restarting
```

**Результат:** `Reboot should not be necessary.` - перезагрузка не нужна. `Reboot is required to fully utilize these updates.` - выполните `sudo reboot` и подключитесь снова. Если команда `needs-restarting` не найдена, установите плагины dnf: `sudo dnf install dnf5-plugins`.

### 2.3. Пользователь с sudo и вход по ключу

Работать под root постоянно не нужно: отдельный пользователь с sudo ограничивает последствия ошибок. Если хостер выдал доступ только под root, создайте пользователя:

```bash
# На VPS под root. Права sudo в Fedora дает группа wheel
useradd -m -G wheel <USER>
# Пароль нужен для sudo
passwd <USER>
```

`useradd` не задает вопросов, пароль задается отдельно командой `passwd`. Если хостер уже создал пользователя (в облачных образах это обычно `fedora`), можно работать под ним: команда `id` должна показать группу `wheel`.

Где выполняется: локальный компьютер с Linux или macOS.

```bash
# Ключ, если его еще нет (файл ~/.ssh/id_ed25519)
ssh-keygen -t ed25519
# Копирование открытого ключа на VPS, пока вход по паролю разрешен
ssh-copy-id <USER>@<IP_VPS>
# Проверка входа по ключу и прав sudo
ssh <USER>@<IP_VPS>
sudo -v
```

В Windows `ssh-keygen` и `ssh` есть в PowerShell, а `ssh-copy-id` нет. Копирование ключа в PowerShell:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh <USER>@<IP_VPS> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && restorecon -R ~/.ssh"
```

`restorecon` восстанавливает метки SELinux на каталоге `~/.ssh`. С неверной меткой sshd не может прочитать `authorized_keys` и отклоняет ключ.

Если хостер уже настроил вход под root по ключу, а вход по паролю отключен, скопируйте ключ root новому пользователю:

```bash
# На VPS под root
install -d -m 700 -o <USER> -g <USER> /home/<USER>/.ssh
install -m 600 -o <USER> -g <USER> /root/.ssh/authorized_keys /home/<USER>/.ssh/authorized_keys
# Метки SELinux для каталога ключей
restorecon -R /home/<USER>/.ssh
```

**Результат:** `ssh <USER>@<IP_VPS>` входит без пароля учетной записи (может запросить только пароль от ключа), `sudo -v` принимает пароль пользователя. `sudo --version` показывает `Sudo version 1.9.17p2`: в Fedora используется классическая реализация sudo. Дальше все команды выполняются от имени `<USER>` через `sudo`.

### 2.4. Вход только по ключу

> **Важно:** выполняйте шаг только после успешной проверки из 2.3. Не закрывайте текущую SSH-сессию, пока не проверите новое подключение.

```bash
sudo tee /etc/ssh/sshd_config.d/10-hardening.conf > /dev/null <<'EOF'
# Вход только по SSH-ключу, вход под root запрещен
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
EOF
# Проверка синтаксиса и применение. В Fedora служба называется sshd
sudo sshd -t && sudo systemctl restart sshd
# Действующие значения
sudo sshd -T | grep -E '^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

**Результат:**

```text
permitrootlogin no
passwordauthentication no
kbdinteractiveauthentication no
```

sshd читает файлы из `/etc/ssh/sshd_config.d/` в алфавитном порядке и использует первое найденное значение параметра. В Fedora там уже есть `40-redhat-crypto-policies.conf` и `50-redhat.conf`, в облачных образах бывает `50-cloud-init.conf` с `PasswordAuthentication yes`. Имя нового файла начинается с `10-`, поэтому он читается раньше.

Проверка с локального компьютера в новом окне терминала:

```bash
# Вход по ключу работает
ssh <USER>@<IP_VPS> true && echo "вход по ключу работает"
# Вход по паролю отклоняется: ожидается "Permission denied (publickey)"
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password <USER>@<IP_VPS>
```

Сообщение может содержать и `gssapi-keyex,gssapi-with-mic`: в Fedora GSSAPI включен в `50-redhat.conf`, без настроенного Kerberos вход через него невозможен.

Откат (при потере доступа - через консоль хостера): `sudo rm /etc/ssh/sshd_config.d/10-hardening.conf && sudo systemctl restart sshd`.

### 2.5. Время

```bash
timedatectl
```

**Результат:** `System clock synchronized: yes` и `NTP service: active`. Неверное время приводит к ошибкам проверки сертификатов. В Fedora время синхронизирует chrony. Если синхронизация выключена:

```bash
sudo systemctl enable --now chronyd
# Состояние chrony: ожидается Leap status: Normal
chronyc tracking
```

Если служба `chronyd` не найдена, установите chrony: `sudo dnf install chrony`.

### 2.6. IP-адреса и IPv6

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

### 2.7. Свободные порты

```bash
sudo ss -tulpn
sudo ss -tulpn | grep -E ':(443|5222|5269|5280|3478|5349)\b'
sudo ss -tulpn | grep -E ':50(0[0-9]{2}|100)\b'
```

**Результат:** две проверки портов пустые. В первом выводе ожидаемы SSH на порту 22, системный DNS-резолвер `systemd-resolved` на `127.0.0.53:53` и `127.0.0.54:53`, chronyd на `127.0.0.1:323`. В Fedora Server порт 9090 занимает `systemd`: это веб-консоль Cockpit (раздел 2.8). `systemd-resolved` может слушать и порт 5355 (LLMNR): снаружи он закрыт файрволом.

Если хостер предустановил веб-сервер и порт 443 занят, остановите его: например, `sudo systemctl disable --now httpd` или `sudo systemctl disable --now nginx`.

### 2.8. Веб-консоль Cockpit

В Fedora Server по умолчанию включена веб-консоль Cockpit на порту 9090, и зона firewalld пропускает к ней подключения. Cockpit принимает вход по паролю пользователя, поэтому через нее можно войти на сервер в обход запрета входа по паролю в SSH (раздел 2.4).

```bash
# enabled - Cockpit запускается при подключении к порту 9090
systemctl is-enabled cockpit.socket
# Отключение
sudo systemctl disable --now cockpit.socket
```

**Результат:** повторный запуск `systemctl is-enabled cockpit.socket` выводит `disabled`. Сообщение `No such file or directory` означает, что Cockpit не установлен (облачные образы): шаг не нужен. Правило firewalld для Cockpit удаляется в разделе 5.

Если Cockpit нужен, включайте его на время работы (`sudo systemctl start cockpit.socket`) и подключайтесь через SSH-туннель, не открывая порт 9090 в файрволе: `ssh -L 9090:127.0.0.1:9090 <USER>@<IP_VPS>` на локальном компьютере, затем `https://localhost:9090` в браузере.

### 2.9. Автоматические обновления безопасности

В Fedora автоматические обновления выполняет плагин dnf5-automatic по таймеру systemd. Настройки по умолчанию только скачивают обновления, поэтому установка обновлений безопасности включается в `/etc/dnf/automatic.conf`. В пакете этот файл пустой, значения по умолчанию лежат в `/usr/share/dnf5/dnf5-plugins/automatic.conf`.

```bash
sudo dnf install dnf5-plugin-automatic
sudo tee /etc/dnf/automatic.conf > /dev/null <<'EOF'
# Автоматическая установка только обновлений безопасности, без перезагрузки.
# Значения по умолчанию: /usr/share/dnf5/dnf5-plugins/automatic.conf
[commands]
upgrade_type = security
apply_updates = yes
reboot = never
EOF
sudo systemctl enable --now dnf5-automatic.timer
systemctl list-timers dnf5-automatic.timer --no-pager
```

**Результат:** `dnf5-automatic.timer` в списке со временем следующего запуска. Таймер срабатывает ежедневно около 6:00 со случайной задержкой до часа. Обновления ядра требуют перезагрузки, а `reboot = never` ее не выполняет: раз в месяц проверяйте необходимость перезагрузки (раздел 11.2).

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
| SRV | `_xmpps-client._tcp` | Priority `0`, Weight `5`, Port `443`, Target `xmpp.inkov.dev` | нет | Auto |
| SRV | `_xmpp-client._tcp` | Priority `10`, Weight `5`, Port `5222`, Target `xmpp.inkov.dev` | нет | Auto |
| SRV | `_xmpp-server._tcp` | Priority `0`, Weight `5`, Port `5269`, Target `xmpp.inkov.dev` | нет | Auto |
| AAAA, только при рабочем IPv6 | `xmpp` | IPv6 address: адрес VPS | DNS only | Auto |
| CNAME, только для Proxy65 | `proxy` | Target: `xmpp.inkov.dev` | DNS only | Auto |

> **Важно:** для записей A, AAAA и CNAME Cloudflare по умолчанию включает проксирование (Proxy status: Proxied, оранжевое облако). Переключите его в **DNS only** (серое облако). Прокси Cloudflare пропускает только HTTP и HTTPS на ограниченном наборе портов, XMPP и TURN через него не работают.

Пояснения к полям:

- **Name** вводится относительно зоны: `xmpp` означает `xmpp.inkov.dev`, `_xmpp-client._tcp` означает `_xmpp-client._tcp.inkov.dev`. Имя зоны Cloudflare добавит сам.
- SRV-записи создаются в зоне `inkov.dev`, а не на поддомене: клиенты ищут их по домену из адреса пользователя. Если форма SRV показывает отдельные поля Service и Protocol, заполните Service `_xmpps-client` (или `_xmpp-client`, `_xmpp-server`), Protocol `TCP`, Name `@`.
- **Priority** задает порядок перебора: меньшее значение пробуется раньше. Клиенты с поддержкой XEP-0368 объединяют записи `_xmpps-client` и `_xmpp-client` и подключаются сначала к порту 443 (приоритет 0), при ошибке - к 5222 (приоритет 10). **Weight** распределяет нагрузку между серверами с одинаковым приоритетом, для одного сервера подходит любое значение.
- **Target** SRV-записи - имя с A-записью (`xmpp.inkov.dev`), а не CNAME: так требует RFC 2782.
- **TTL Auto** в Cloudflare равен 300 секундам.
- Для записей в режиме DNS only панель может предупредить, что IP-адрес будет виден. Для этой схемы это ожидаемо.
- Существующие A-записи `inkov.dev` (GitHub Pages) и запись `www`, если она есть, не меняйте.

### 3.2. CAA-записи

CAA-записи ограничивают список центров сертификации, которые могут выпускать сертификаты для домена.

```bash
# Утилита dig (один раз)
sudo dnf install bind-utils
dig +short CAA inkov.dev
```

- Пустой вывод - ограничений нет, дальше ничего делать не нужно.
- Если записи есть, среди них должна быть `0 issue "letsencrypt.org"`. Если есть записи с тегом `issuewild`, нужна и `0 issuewild "letsencrypt.org"`, иначе wildcard-сертификат не будет выпущен. Недостающую запись добавьте в Cloudflare: Type `CAA`, Name `@`, тег выбирается в поле **Tag**, значение `letsencrypt.org`.

GitHub Pages тоже получает сертификаты Let's Encrypt, поэтому существующие CAA-записи обычно уже разрешают `letsencrypt.org`.

### 3.3. Проверка

Запросы напрямую к DNS-серверу Cloudflare показывают записи сразу после сохранения, без кэша промежуточных резолверов.

```bash
# DNS-сервер Cloudflare для зоны
NS=$(dig +short NS inkov.dev | head -n 1); echo "$NS"
dig +short @"$NS" A xmpp.inkov.dev
dig +short @"$NS" SRV _xmpps-client._tcp.inkov.dev
dig +short @"$NS" SRV _xmpp-client._tcp.inkov.dev
dig +short @"$NS" SRV _xmpp-server._tcp.inkov.dev
dig +short @"$NS" upload.inkov.dev
```

**Результат** (с адресом `203.0.113.10` из примера):

```text
xxxx.ns.cloudflare.com.
203.0.113.10
0 5 443 xmpp.inkov.dev.
10 5 5222 xmpp.inkov.dev.
0 5 5269 xmpp.inkov.dev.
xmpp.inkov.dev.
203.0.113.10
```

Затем проверьте ответ обычного резолвера:

```bash
dig +short SRV _xmpps-client._tcp.inkov.dev
```

Ответ должен совпасть с предыдущим. Если он пустой, подождите: резолвер, который уже запрашивал это имя до создания записи, хранит отрицательный ответ до 30 минут.

---

## 4. Установка Prosody

### 4.1. Пакет Prosody в Fedora

```bash
# Версия и репозиторий пакета
dnf info prosody | grep -E '^(Name|Version|Release|Repository)'
```

**Результат:**

```text
Name           : prosody
Version        : 13.0.6
Release        : 1.fc44
Repository     : updates
```

Номер версии может быть новее: Fedora обновляет пакет в пределах ветки 13.x. Строка `Repository : fedora` означает, что обновления еще не загружены: выполните раздел 2.2.

### 4.2. Установка

```bash
sudo dnf install prosody lua-dbi
```

Пакеты:

- `prosody` - сервер. dnf ставит вместе с ним рекомендуемые пакеты `lua-unbound` (DNS-резолвер) и `lua-readline` (редактирование строк в `prosodyctl shell`): в Fedora рекомендуемые зависимости устанавливаются по умолчанию.
- `lua-dbi` - драйвер LuaDBI, через него Prosody работает с базой SQLite. В одном пакете собраны драйверы SQLite, MySQL и PostgreSQL, поэтому dnf ставит и клиентские библиотеки `mariadb-connector-c` и `libpq`. Prosody использует только SQLite.
- `luarocks` не нужен. Он требуется только для установки сторонних модулей командой `prosodyctl install` и тянет за собой компилятор gcc.

Проверка:

```bash
sudo prosodyctl about | grep -E '^Prosody|Lua version|LuaRocks|LuaDBI|luaunbound'
# Драйвер SQLite загружается в Lua 5.4
lua -e 'require "DBI"; print("LuaDBI: ok")'
# Служба после установки не включена
systemctl is-enabled prosody
```

**Результат:**

```text
Prosody 13.0.6
Lua version:             	Lua 5.4
LuaRocks:        	Not installed
LuaDBI:       	0.7
luaunbound:   	1.1.0
LuaDBI: ok
disabled
```

Если строки `luaunbound` нет, установите пакеты вручную: `sudo dnf install lua-unbound lua-readline`. Ошибка `module 'DBI' not found` означает, что не установлен `lua-dbi`. Служба Prosody пока не запущена, она включается в разделе 9.1 после настройки. Пакет также создал самоподписанный сертификат для `localhost` в `/etc/pki/prosody`, в этой схеме он не используется.

### 4.3. Право на порт 443: systemd и SELinux

Prosody не сможет слушать порт 443 без двух изменений:

1. Служба `prosody` из пакета запускается от пользователя `prosody` без права слушать порты ниже 1024. Без него в журнале будет ошибка привязки к порту 443 с `Permission denied`. Право добавляется параметром `AmbientCapabilities` в файле drop-in: файл пакета `/usr/lib/systemd/system/prosody.service` не меняется и не перезаписывается при обновлении.
2. Политика SELinux разрешает Prosody порты XMPP (5222, 5269) и 5280-5281. Порт 443 имеет тип `http_port_t`, его политика разрешает только при включенном булевом параметре `prosody_bind_http_port`. По умолчанию параметр выключен.

```bash
# Drop-in для службы prosody
sudo install -d -m 755 /etc/systemd/system/prosody.service.d
sudo tee /etc/systemd/system/prosody.service.d/10-bind-443.conf > /dev/null <<'EOF'
# Право слушать порты ниже 1024 (443) без запуска от root (раздел 4.3)
[Service]
AmbientCapabilities=CAP_NET_BIND_SERVICE
EOF
sudo systemctl daemon-reload
# Действующее значение
systemctl show prosody -p AmbientCapabilities

# SELinux: Prosody может слушать порты типа http_port_t. Ключ -P сохраняет значение после перезагрузки
sudo setsebool -P prosody_bind_http_port on
getsebool prosody_bind_http_port
```

**Результат:**

```text
AmbientCapabilities=cap_net_bind_service
prosody_bind_http_port --> on
```

`setsebool -P` пересобирает политику и может выполняться до минуты. Тип `http_port_t` имеют и другие порты (80, 8443 и несколько других), но Prosody слушает только порты из своей конфигурации.

Откат:

```bash
sudo rm -r /etc/systemd/system/prosody.service.d && sudo systemctl daemon-reload
sudo setsebool -P prosody_bind_http_port off
```

---

## 5. Файрвол

В Fedora Server firewalld включен после установки системы. Правила добавляются в зону по умолчанию.

> **Важно:** не закрывайте текущую SSH-сессию, пока не проверите новое подключение после изменения правил.

```bash
# Состояние firewalld: ожидается running
sudo firewall-cmd --state
# Зона по умолчанию и ее правила
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --list-all
```

**Результат:** `running`, зона `FedoraServer` (в облачных образах и после ручной установки firewalld - `public`). В строке `services:` есть `ssh`, в Fedora Server также `cockpit` и `dhcpv6-client`.

Если `firewall-cmd` не найден или сообщает `not running`, установите и запустите firewalld. В зоне `public`, которая используется по умолчанию, SSH на порту 22 разрешен, поэтому текущая сессия не прервется.

```bash
sudo dnf install firewalld
# Только если SSH работает не на порту 22: разрешить его порт до запуска firewalld
# sudo firewall-offline-cmd --add-port=<PORT>/tcp
sudo systemctl enable --now firewalld
```

Правила для Prosody:

```bash
# XMPP: прямой TLS и HTTPS (443), STARTTLS (5222), связь с другими серверами (5269)
sudo firewall-cmd --permanent --add-service=https --add-service=xmpp-client --add-service=xmpp-server
# Веб-консоль Cockpit снаружи не нужна (раздел 2.8)
sudo firewall-cmd --permanent --remove-service=cockpit
# Применение постоянных правил
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```

**Результат:** каждая команда выводит `success`, последняя - список служб:

```text
dhcpv6-client https ssh xmpp-client xmpp-server
```

В зоне `public` в списке есть и `mdns`. Если Cockpit в зоне не было, удаление выведет `Warning: NOT_ENABLED: cockpit`, это ожидаемо. Затем откройте новое SSH-подключение. Если доступ пропал, в консоли хостера выполните `sudo systemctl stop firewalld`.

- Службы firewalld - наборы портов: `https` - 443/tcp, `xmpp-client` - 5222/tcp, `xmpp-server` - 5269/tcp. Описания лежат в `/usr/lib/firewalld/services/`.
- `--permanent` записывает правило в конфигурацию, `--reload` применяет ее. Правило без `--permanent` действует до перезагрузки firewalld.
- Порт 5280 не открывается: он нужен только локально. Порт 80 не нужен: сертификат выпускается через DNS.
- Порты coturn (3478 и 50000-50100/udp) открываются в разделе 8.3, после настройки coturn. Порты 5000 (Proxy65) и 5349 (TURN с TLS) открываются только при включении этих функций (разделы 7.3 и 8.4).
- Правила действуют для IPv4 и IPv6.
- Если в панели хостера включен внешний файрвол, откройте в нем те же порты.
- Удаление правила: та же команда с `--remove-service` или `--remove-port`, затем `sudo firewall-cmd --reload`.

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

Токен вставляется в редакторе и не попадает в историю shell. Если nano не установлен: `sudo dnf install nano`.

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
sudo dnf install certbot python3-certbot-dns-cloudflare
certbot --version
```

**Результат:** `certbot 5.8.0` или новее. Это версия из репозитория Fedora 44, она поддерживает DNS-01 через API Cloudflare с токеном. Snap-версию certbot параллельно не ставьте: обе установки используют каталог `/etc/letsencrypt` и запускают продление независимо друг от друга.

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

`--deploy-hook` выполняется после каждого успешного выпуска и продления. `prosodyctl --root cert import` копирует сертификат и ключ в каталог сертификатов Prosody (`/etc/pki/prosody`) с владельцем `prosody` и перезагружает Prosody. Указывать Prosody пути к файлам в `/etc/letsencrypt` напрямую нельзя: ключ там доступен только root, а Prosody работает от пользователя `prosody`.

**Результат:** в выводе есть строки

```text
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/inkov.dev/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/inkov.dev/privkey.pem
```

При первом выпуске certbot также сообщит, что команда импорта завершилась с ошибкой (`reported error code 1`), а в ее выводе будут строки `No certificate for host localhost found :(` и `No certificates imported :(`. Это ожидаемо: в конфигурации Prosody по умолчанию есть только хост `localhost`, для которого в сертификате нет имени. Сертификат при этом сохранен, команда импорта записана в настройки продления. Импорт выполняется в разделе 9.2, после настройки Prosody.

Проверка:

```bash
sudo certbot certificates
```

**Результат:** `Domains: inkov.dev *.inkov.dev`, `Expiry Date` примерно через 90 дней.

### 6.7. Автоматическое продление

certbot из пакета Fedora устанавливает таймер `certbot-renew.timer`, но не включает его. Таймер запускает `certbot renew` дважды в сутки со случайной задержкой. certbot продлевает сертификат заранее, для 90-дневного сертификата - примерно за 30 дней до окончания срока. После продления выполняется `--deploy-hook`.

```bash
# Включение таймера продления
sudo systemctl enable --now certbot-renew.timer
systemctl list-timers certbot-renew.timer --no-pager
# Команда импорта сохранена в настройках продления
sudo grep renew_hook /etc/letsencrypt/renewal/inkov.dev.conf
# Проверка продления на тестовом сервере
sudo certbot renew --dry-run
```

**Результат:** `certbot-renew.timer` в списке со временем следующего запуска; строка `renew_hook = prosodyctl --root cert import /etc/letsencrypt/live`; сообщение `Congratulations, all simulated renewals succeeded`.

Служба `certbot-renew.service` читает дополнительные параметры из `/etc/sysconfig/certbot`. Оставьте переменные `PRE_HOOK`, `POST_HOOK` и `DEPLOY_HOOK` в этом файле пустыми: значение `DEPLOY_HOOK` заменяет команду импорта, сохраненную для сертификата.

При `--dry-run` deploy hook по умолчанию не выполняется. Проверка продления вместе с командой импорта - в разделе 9.2, после настройки Prosody.

> **Важно:** с июня 2025 года Let's Encrypt не рассылает письма об истечении сертификатов. Если продление перестанет работать (например, API Cloudflare станет недоступен с VPS), уведомления не будет. Раз в месяц проверяйте срок командой `sudo certbot certificates` или настройте внешний мониторинг сертификата по адресу `https://upload.inkov.dev/`.

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
dig +short @"$(dig +short NS inkov.dev | head -n 1)" TXT _acme-challenge.inkov.dev
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

Как устроена конфигурация в пакете Fedora:

- Основной файл `/etc/prosody/prosody.cfg.lua` в конце подключает файлы `conf.d/*.cfg.lua` строкой `Include`. В `conf.d` лежат примеры `localhost.cfg.lua` и `example.com.cfg.lua`.
- `/etc/prosody/certs` - символическая ссылка на `/etc/pki/prosody`, где хранятся сертификаты.

Новая конфигурация хранится целиком в основном файле и не подключает `conf.d`, поэтому примеры из пакета не используются. Удалять их не нужно.

### 7.2. Секрет для TURN

```bash
sudo install -d -m 700 /root/.secrets
sudo sh -c 'umask 077; openssl rand -hex 32 > /root/.secrets/turn_secret'
```

Секрет подставляется в конфигурации Prosody (7.4) и coturn (8.2) командой `sed`, копировать его вручную не нужно.

### 7.3. Назначение настроек

| Параметр | Значение | Назначение |
|---|---|---|
| `admins` | `admin@inkov.dev`, `tzx1z@inkov.dev` | администраторы сервера: команды из клиента, приглашения. Учетные записи создаются отдельно (раздел 10.1) |
| `modules_enabled` | список из пакета, плюс `mam`, `turn_external` и `net_multiplex` | набор функций сервера, пояснения ниже |
| `ssl_ports` | `443` | порт с TLS, на котором `net_multiplex` принимает прямой TLS для клиентов и HTTPS |
| `https_ports` | пустой список | отдельный порт HTTPS не нужен: HTTPS работает через 443 |
| `http_interfaces` | `127.0.0.1` | нешифрованный HTTP на порту 5280 доступен только с самого сервера |
| `http_external_url` | `https://upload.inkov.dev/` (в компоненте обмена файлами) | адрес в ссылках на файлы. Без него Prosody подставит адрес нешифрованного порта 5280 |
| `allow_registration` | `false` | регистрация из клиентов закрыта. Учетные записи создает администратор или они создаются по приглашению (модули `invites*`) |
| `c2s_require_encryption`, `s2s_require_encryption` | `true` | подключение без TLS невозможно |
| `s2s_secure_auth` | `true` | другие серверы должны предъявить действительный сертификат |
| `limits` | 10 и 30 КБ/с | ограничение входящего трафика от клиентов и серверов, значения из пакета |
| `pidfile` | `/run/prosody/prosody.pid` | по нему `prosodyctl` перезагружает сервер, в том числе после импорта сертификата. Путь из пакета Fedora |
| `authentication` | `internal_hashed` | пароли хранятся в виде хешей SCRAM |
| `storage`, `sql` | `sql`, драйвер `SQLite3`, файл `prosody.sqlite` | все данные Prosody в одной базе `/var/lib/prosody/prosody.sqlite`. Таблицы Prosody создает сам при первом запуске. Загруженные файлы хранятся на диске |
| `archive_expires_after` | `1y` | история хранится на сервере год и синхронизируется между устройствами. Меньший срок - меньше данных на сервере. Примеры значений: `1w`, `30d`, `6 months`, `1y`, `never`. Значение `1m` неоднозначно (месяц или минута), Prosody 13 записывает его в журнал как ошибку |
| `turn_external_*` | `turn.inkov.dev`, секрет, TCP | клиенты получают адрес coturn и временные учетные данные, действующие сутки |
| `certificates` | `/etc/pki/prosody` | каталог сертификатов из пакета Fedora. Prosody сам находит в нем сертификат для каждого хоста, для поддоменов - через wildcard |

Как Prosody делит порт 443: `net_multiplex` завершает TLS со своим сертификатом (по SNI) и выбирает сервис по ALPN: `xmpp-client` - подключение клиента, `http/1.1` - HTTPS. Если клиент ALPN не передал, сервис определяется по первым байтам: поток XMPP или HTTP-запрос. Связь с другими серверами на 443 не используется (раздел 1.3).

Модули, от которых зависит работа мобильных клиентов:

- `smacks` - восстановление сессии после смены сети без потери сообщений;
- `carbons` - копии сообщений на все устройства пользователя;
- `csi_simple` - отложенная доставка второстепенного трафика, пока приложение в фоне;
- `cloud_notify` - push-уведомления (XEP-0357), в Prosody 13 входит в поставку и включен по умолчанию;
- `mam` - архив сообщений на сервере.

Компоненты:

- `conference.inkov.dev` (`muc`) - групповые чаты. `muc_mam` хранит историю комнат, `restrict_room_creation = "local"` разрешает создавать комнаты только пользователям `inkov.dev`.
- `upload.inkov.dev` (`http_file_share`) - обмен файлами. Это отдельный компонент, а не модуль из `modules_enabled`. Лимиты: файл до 100 МБ, до 1 ГБ в сутки на пользователя, до 5 ГБ на сервере, хранение 30 дней. Большие файлы записываются на диск потоком, параметр `http_max_content_size` менять не нужно.
- `proxy.inkov.dev` (`proxy65`) - прямая передача файлов для старых клиентов. Современные клиенты используют `http_file_share`, поэтому компонент закомментирован. Для включения уберите `--` в двух строках, создайте DNS-запись `proxy` (раздел 3.1) и откройте порт: `sudo firewall-cmd --permanent --add-port=5000/tcp && sudo firewall-cmd --reload`. Политика SELinux разрешает Prosody порт 5000 без дополнительных настроек.

### 7.4. Файл конфигурации

Скопируйте блок целиком и выполните его в терминале: команда заменит содержимое файла.

```bash
sudo tee /etc/prosody/prosody.cfg.lua > /dev/null <<'EOF'
-- /etc/prosody/prosody.cfg.lua
-- Личный XMPP-сервер для адресов вида user@inkov.dev.
-- Основа - конфигурация из пакета Prosody 13.0 в Fedora 44.
-- После каждой правки: sudo prosodyctl check config

---------- Глобальные настройки ----------
-- Действуют на весь сервер. Должны стоять выше первой строки VirtualHost или Component.

-- Администраторы. Учетную запись нужно создать отдельно командой prosodyctl adduser.
admins = { "admin@inkov.dev", "tzx1z@inkov.dev" }

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

    -- Сеть
        "net_multiplex"; -- несколько сервисов на одном порту (443)
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
pidfile = "/run/prosody/prosody.pid"

-- Пароли хранятся в виде хешей (SCRAM)
authentication = "internal_hashed"

-- Хранилище: база SQLite в одном файле. Нужен драйвер из пакета lua-dbi (раздел 4.2).
-- Таблицы Prosody создает сам при первом запуске.
storage = "sql"
sql = {
    driver = "SQLite3";
    database = "prosody.sqlite"; -- путь относительно /var/lib/prosody
}

-- Срок хранения архива личных сообщений
archive_expires_after = "1y"

-- Порт 443: прямой TLS для клиентов (XEP-0368) и HTTPS для обмена файлами.
-- Prosody выбирает сервис по ALPN, а если клиент ALPN не передал - по первым байтам.
ssl_ports = { 443 }
-- Отдельный порт HTTPS не нужен: HTTPS работает через порт 443
https_ports = { }
-- Нешифрованный HTTP (5280) доступен только с самого сервера
http_ports = { 5280 }
http_interfaces = { "127.0.0.1" }

-- TURN-сервер для звонков (coturn). Секрет совпадает с static-auth-secret в /etc/coturn/turnserver.conf.
-- Если звонки не нужны, удалите эти три строки и "turn_external" из modules_enabled.
turn_external_host = "turn.inkov.dev"
turn_external_secret = "<TURN_SECRET>"
turn_external_tcp = true

-- Журналы
log = {
    info = "/var/log/prosody/prosody.log"; -- замените info на debug для подробного журнала
    error = "/var/log/prosody/prosody.err";
}

-- Каталог сертификатов. В Fedora /etc/prosody/certs - ссылка на этот каталог.
certificates = "/etc/pki/prosody"

----------- Виртуальный хост -----------
-- Домен в адресах пользователей: user@inkov.dev
VirtualHost "inkov.dev"

------------- Компоненты -------------

-- Групповые чаты: комнаты вида room@conference.inkov.dev
Component "conference.inkov.dev" "muc"
    modules_enabled = { "muc_mam" } -- архив сообщений в комнатах
    restrict_room_creation = "local" -- создавать комнаты могут только пользователи inkov.dev
    muc_log_expires_after = "1y" -- срок хранения архива комнат

-- Обмен файлами: https://upload.inkov.dev/ (порт 443)
Component "upload.inkov.dev" "http_file_share"
    http_external_url = "https://upload.inkov.dev/" -- адрес в ссылках на файлы
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
# Владелец root, группа prosody, остальным доступа нет (так же в пакете)
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

Сообщение про `check features` относится к подключениям из браузера (раздел 9.3) и на работу сервера не влияет. Новая конфигурация применяется при запуске в разделе 9.1.

---

## 8. Звонки: coturn

Если звонки не нужны, пропустите раздел. Тогда удалите из `/etc/prosody/prosody.cfg.lua` строку `"turn_external";` и три строки `turn_external_*`.

### 8.1. Установка

```bash
sudo dnf install coturn
# После установки служба не запущена: ожидается inactive
systemctl is-active coturn
```

> **Важно:** с файлом конфигурации из пакета coturn выделяет relay-порты без проверки учетных данных. Поэтому служба запускается только после настройки, а порты TURN в файрволе открываются после запуска с новой конфигурацией (раздел 8.3).

### 8.2. Конфигурация

```bash
# Резервная копия файла из пакета
sudo cp /etc/coturn/turnserver.conf /etc/coturn/turnserver.conf.orig
sudo tee /etc/coturn/turnserver.conf > /dev/null <<'EOF'
# /etc/coturn/turnserver.conf - TURN-сервер для звонков через XMPP

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

# TLS не используется (см. раздел 8.4). DTLS в coturn 4.18 выключен по умолчанию.
no-tls

# Безопасность. Консоль управления (CLI) и атрибут SOFTWARE с версией coturn
# в coturn 4.18 выключены по умолчанию.
fingerprint
no-multicast-peers

# Запрет пересылки на локальные, частные и служебные адреса.
# Без этого через TURN можно обратиться к сервисам, которые слушают только 127.0.0.1.
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

# Журнал в файле, как в конфигурации из пакета. Ротацию выполняет logrotate.
# С параметром syslog вместо файла coturn 4.18 завершается по сигналу HUP,
# который отправляет systemctl reload coturn.
log-file=/var/log/coturn/turnserver.log
simple-log
EOF
# Секрет TURN из файла вместо <TURN_SECRET>
sudo sh -c 'sed -i "s/<TURN_SECRET>/$(cat /root/.secrets/turn_secret)/" /etc/coturn/turnserver.conf'
# Файл читают только root и группа coturn (так же в пакете)
sudo chown root:coturn /etc/coturn/turnserver.conf
sudo chmod 640 /etc/coturn/turnserver.conf
```

Если в разделе 2.6 выяснилось, что VPS работает за NAT, раскомментируйте строку `external-ip` и укажите публичный и внутренний адреса: `sudo nano /etc/coturn/turnserver.conf`.

Назначение параметров:

- `use-auth-secret` и `static-auth-secret` - Prosody выдает клиентам временные логин и пароль, вычисленные из общего секрета. Постоянные пароли TURN не нужны.
- `min-port` и `max-port` - 101 порт для медиа. Один звонок через TURN занимает несколько портов, для личного сервера этого достаточно.
- `denied-peer-ip` - запрет пересылки на внутренние адреса. Без него пользователь TURN может обращаться к сервисам, которые слушают только `127.0.0.1` или внутреннюю сеть хостера.
- `log-file` и `simple-log` - журнал `/var/log/coturn/turnserver.log`, как в конфигурации из пакета. Пакет настраивает его ротацию через `systemctl try-reload-or-restart coturn`.
- В coturn 4.18 консоль управления, DTLS и атрибут SOFTWARE с версией выключены по умолчанию, минимальная версия TLS - 1.2. Параметры `no-cli`, `no-dtls`, `no-software-attribute`, `no-tlsv1` и `no-tlsv1_1` из конфигураций для старых версий не нужны: на `no-dtls` coturn 4.18 выводит `Bad configuration format`, на `no-cli` - сообщение об устаревшем параметре.

SELinux на coturn не влияет: в политике Fedora нет отдельного модуля для coturn.

### 8.3. Запуск и проверка

```bash
# Запуск и автозапуск при загрузке
sudo systemctl enable --now coturn
systemctl status coturn --no-pager
sudo ss -tulpn | grep turnserver
# Предупреждения при запуске
sudo grep -E 'WARNING|ERROR' /var/log/coturn/turnserver.log
```

**Результат:** `Active: active (running)`, `turnserver` слушает порт 3478 по UDP и TCP. В журнале ожидаемы только такие предупреждения (в начале каждой строки дата и время):

```text
WARNING Certificate file not found or not readable: //turn_server_cert.pem
WARNING Private key file not found or not readable: //turn_server_pkey.pem
WARNING NO EXPLICIT LISTENER ADDRESS(ES) ARE CONFIGURED
WARNING NO EXPLICIT RELAY ADDRESS(ES) ARE CONFIGURED
```

Сертификат не нужен, пока TLS выключен. Без явных адресов coturn слушает все адреса сервера.

Порты TURN в файрволе: служба `stun` (3478/tcp и 3478/udp) и диапазон передачи медиа.

```bash
sudo firewall-cmd --permanent --add-service=stun --add-port=50000-50100/udp
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
sudo firewall-cmd --list-ports
```

**Результат:** в списке служб есть `stun`, в списке портов - `50000-50100/udp`. Если у хостера есть внешний файрвол, откройте те же порты в нем.

Проверка связки Prosody и coturn. Команда читает настройки `turn_external_*` из конфигурации Prosody, запуск Prosody для нее не нужен. Имя `turn.inkov.dev` должно уже разрешаться (раздел 3).

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

TURN поверх TLS на порту 5349 помогает клиентам в сетях, где закрыт UDP, но разрешены TCP-соединения с TLS. Порт 443 для этого использовать нельзя: он занят Prosody.

Пакет coturn создает каталоги для сертификатов: `/etc/pki/coturn/public` (открыт для чтения всем) и `/etc/pki/coturn/private` (только root и группа `coturn`). Скрипт ниже копирует в них сертификат. Скрипты из `/etc/letsencrypt/renewal-hooks/deploy` certbot выполняет после каждого продления.

```bash
sudo install -d /etc/letsencrypt/renewal-hooks/deploy
sudo tee /etc/letsencrypt/renewal-hooks/deploy/coturn.sh > /dev/null <<'EOF'
#!/bin/sh
# Копирует сертификат Let's Encrypt для coturn. coturn перечитывает сертификат
# по systemctl reload, текущие звонки не прерываются.
set -e
install -o root -g root -m 644 /etc/letsencrypt/live/inkov.dev/fullchain.pem /etc/pki/coturn/public/fullchain.pem
install -o root -g coturn -m 640 /etc/letsencrypt/live/inkov.dev/privkey.pem /etc/pki/coturn/private/privkey.pem
systemctl try-reload-or-restart coturn
EOF
sudo chmod 755 /etc/letsencrypt/renewal-hooks/deploy/coturn.sh
```

Включение TLS в coturn и Prosody:

```bash
# Первое копирование сертификата
sudo /etc/letsencrypt/renewal-hooks/deploy/coturn.sh
# coturn: убрать запрет TLS и добавить порт и сертификат
sudo sed -i '/^no-tls$/d' /etc/coturn/turnserver.conf
sudo tee -a /etc/coturn/turnserver.conf > /dev/null <<'EOF'

# TLS (раздел 8.4). Минимальная версия TLS в coturn 4.18 - 1.2.
tls-listening-port=5349
cert=/etc/pki/coturn/public/fullchain.pem
pkey=/etc/pki/coturn/private/privkey.pem
EOF
# Новый порт coturn открывает только при перезапуске
sudo systemctl restart coturn
# Prosody: сообщать клиентам порт TURN с TLS
sudo sed -i 's/^turn_external_tcp = true$/&\nturn_external_tls_port = 5349/' /etc/prosody/prosody.cfg.lua
# Файрвол: только TCP, DTLS не используется
sudo firewall-cmd --permanent --add-port=5349/tcp
sudo firewall-cmd --reload
```

Если Prosody уже запущен (раздел 9.1), выполните `sudo systemctl restart prosody`: `mod_turn_external` читает параметры при загрузке модуля, `reload` новый порт не применит.

Проверка:

```bash
openssl s_client -connect turn.inkov.dev:5349 </dev/null 2>/dev/null | openssl x509 -noout -enddate -ext subjectAltName
```

**Результат:** строка с датой окончания и строка с `DNS:inkov.dev, DNS:*.inkov.dev`. При продлении сертификата coturn перечитывает его по сигналу, текущие звонки не прерываются.

---

## 9. Запуск и проверка

### 9.1. Запуск Prosody

```bash
# Запуск и автозапуск при загрузке
sudo systemctl enable --now prosody
systemctl status prosody --no-pager
# Последние записи журнала и файл ошибок
sudo tail -n 30 /var/log/prosody/prosody.log
sudo cat /var/log/prosody/prosody.err
```

**Результат:** `Active: active (running)`. В журнале есть строки:

```text
portmanager	info	Activated service 's2s' on [::]:5269, [*]:5269
portmanager	info	Activated service 'multiplex_ssl' on [::]:443, [*]:443
portmanager	info	Activated service 'c2s' on [::]:5222, [*]:5222
portmanager	info	Activated service 'http' on [127.0.0.1]:5280
portmanager	info	Activated service 'https' on no ports
upload.inkov.dev:http	info	Serving 'file_share' at https://upload.inkov.dev/file_share
```

Порт 443 слушает `multiplex_ssl`, строка `'https' on no ports` ожидаема: HTTPS работает через 443. `prosody.err` пустой или отсутствует. Если служба не запустилась, причина видна в `journalctl -u prosody -n 50 --no-pager`.

Если в журнале ошибка привязки к порту 443 с `Permission denied`, проверьте оба изменения из раздела 4.3 и отказы SELinux:

```bash
systemctl show prosody -p AmbientCapabilities
getsebool prosody_bind_http_port
# Отказы SELinux за последние 10 минут: ожидается <no matches>
sudo ausearch -m avc -ts recent
```

Проверка базы SQLite:

```bash
sudo ls -l /var/lib/prosody/prosody.sqlite
sudo grep -c -E 'LuaDBI or LuaSQLite3|no data storage' /var/log/prosody/prosody.err
```

**Результат:** файл `prosody.sqlite` с владельцем `prosody` и правами `-rw-r-----`, счетчик ошибок `0`. Если файла нет или счетчик больше нуля, Prosody работает без хранилища: проверьте драйвер (раздел 4.2).

### 9.2. Импорт сертификата и тест продления

Эту же команду certbot выполняет после каждого продления. Теперь в конфигурации есть хост `inkov.dev` и компоненты, поэтому импорт проходит:

```bash
sudo prosodyctl --root cert import /etc/letsencrypt/live
sudo ls -l /etc/pki/prosody/
```

**Результат:** `Imported certificate and key for hosts inkov.dev, upload.inkov.dev, conference.inkov.dev`. В `/etc/pki/prosody` появились файлы `inkov.dev.crt` и `inkov.dev.key` с владельцем `prosody`. Prosody перечитывает сертификаты автоматически, в журнале появляются строки `Certificates reloaded`.

Тест продления вместе с командой импорта:

```bash
sudo certbot renew --dry-run --run-deploy-hooks
```

**Результат:** `Congratulations, all simulated renewals succeeded` и сообщение команды импорта `Imported certificate and key for hosts ...`. Если включен TLS для coturn (8.4), выполняется и скрипт `coturn.sh`.

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
| `check dns` | SRV-записи, A и AAAA-записи сервера и компонентов, совпадение адресов с IP VPS | только ожидаемые сообщения про `_xmpps-server` (ниже) |
| `check certs` | наличие сертификатов для `inkov.dev` и компонентов, имена и срок действия | для каждого хоста `Certificate: /etc/pki/prosody/inkov.dev.crt` |
| `check turn -v` | выдачу учетных данных TURN и выделение relay-порта | `Success!` |
| `check connectivity` | доступность портов 5222 и 5269 из интернета через сервис observe.jabber.network | `xmpp-client: Works`, `xmpp-server: Works` |
| `check features` | набор функций для клиентов | все пункты `OK`, кроме `Web connections` |

Особенности этой схемы:

- С `net_multiplex` проверка `check dns` считает, что порт 443 обслуживает и связь с другими серверами, и выводит `No _xmpps-server SRV record found for inkov.dev, but it looks like you need one.`, а также такие же строки для `conference.inkov.dev` и `upload.inkov.dev`. Эти сообщения ожидаемы: связь с другими серверами работает через 5269 (раздел 1.3). Других сообщений быть не должно.
- `check dns` проверяет цели SRV-записей. Сообщение `inkov.dev A record points to unknown address 185.199.x.x` означает, что SRV-записи не видны серверу: вернитесь к разделу 3.3.
- `check connectivity` не проверяет порт 443, его проверка - в разделе 9.4. Если внешний сервис недоступен из сети хостера, сообщение `Failed to request check at API` не означает проблем с сервером.
- `check features` отмечает `(!) Web connections`: это BOSH и WebSocket для браузерных клиентов. Мобильным и настольным клиентам они не нужны.
- Если VPS работает за NAT, `check dns` сообщает, что адреса из DNS не найдены на сервере. Укажите публичный адрес в глобальной части конфигурации, выше строки `VirtualHost`: `external_addresses = { "<IP_VPS>" }`.

### 9.4. Проверка снаружи

Где выполняется: локальный компьютер с Linux или macOS. В Windows порты проверяются в PowerShell: `Test-NetConnection xmpp.inkov.dev -Port 443`, ожидается `TcpTestSucceeded : True`.

```bash
# SRV-записи через обычный резолвер
dig +short SRV _xmpps-client._tcp.inkov.dev
dig +short SRV _xmpp-client._tcp.inkov.dev
dig +short SRV _xmpp-server._tcp.inkov.dev

# Доступность портов
nc -vz xmpp.inkov.dev 443
nc -vz xmpp.inkov.dev 5222
nc -vz xmpp.inkov.dev 5269

# Прямой TLS на 443: Prosody отвечает списком механизмов входа
(printf "<?xml version='1.0'?><stream:stream to='inkov.dev' xmlns='jabber:client' xmlns:stream='http://etherx.jabber.org/streams' version='1.0'>"; sleep 3) \
  | timeout 6 openssl s_client -connect xmpp.inkov.dev:443 -servername inkov.dev -alpn xmpp-client -quiet 2>/dev/null \
  | grep -o '<mechanism>[A-Z0-9-]*</mechanism>'
# HTTPS для обмена файлами на 443
curl -s -o /dev/null -w '%{http_code}\n' https://upload.inkov.dev/file_share/
# Сертификаты: должны содержать inkov.dev
openssl s_client -connect xmpp.inkov.dev:443 -servername inkov.dev </dev/null 2>/dev/null \
  | openssl x509 -noout -enddate -text | grep -E 'notAfter=|DNS:'
openssl s_client -connect xmpp.inkov.dev:5222 -starttls xmpp -xmpphost inkov.dev </dev/null 2>/dev/null \
  | openssl x509 -noout -enddate -text | grep -E 'notAfter=|DNS:'
openssl s_client -connect xmpp.inkov.dev:5269 -starttls xmpp-server -xmpphost inkov.dev </dev/null 2>/dev/null \
  | openssl x509 -noout -enddate -text | grep -E 'notAfter=|DNS:'
```

**Результат:**

- SRV-записи как в разделе 3.3;
- `nc` сообщает об успешном подключении (`Connected to` или `succeeded`) для всех трех портов;
- строки `<mechanism>SCRAM-SHA-1</mechanism>`, `<mechanism>SCRAM-SHA-1-PLUS</mechanism>`, `<mechanism>PLAIN</mechanism>` и `<mechanism>OAUTHBEARER</mechanism>`;
- `curl` выводит `200`;
- для каждого порта дата окончания сертификата и строка с `DNS:inkov.dev` и `DNS:*.inkov.dev`.

### 9.5. Внешние сервисы проверки

- https://compliance.conversations.im проверяет поддержку расширений XMPP, важных для мобильных клиентов. Сервис входит на сервер под учетной записью, поэтому используйте временную `test@inkov.dev` (раздел 10.1) и удалите ее после проверки.
- Доступность внешних сервисов из России и сети хостера может меняться. Основная проверка - команды из 9.3 и 9.4.

---

## 10. Пользователи и клиенты

### 10.1. Учетные записи

```bash
# Администраторы (адреса указаны в admins). Пароль вводится дважды и не попадает в историю shell
sudo prosodyctl adduser admin@inkov.dev
sudo prosodyctl adduser tzx1z@inkov.dev
# Тестовый пользователь
sudo prosodyctl adduser test@inkov.dev
# Список пользователей
sudo prosodyctl shell user list inkov.dev
# Фактическая роль администратора
sudo prosodyctl shell user role admin@inkov.dev
```

**Результат:** `OK: Created admin@inkov.dev with role 'prosody:member'` - роль при создании. Права администратора дает список `admins`, поэтому `user role` показывает `OK: prosody:operator`. Учетные записи хранятся в базе `/var/lib/prosody/prosody.sqlite`. Ошибка `Could not create user: no data storage active` означает, что хранилище не работает (раздел 12).

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
- Сервер и порт указывать не нужно: клиент находит их по SRV-записям. Клиенты с поддержкой XEP-0368 подключаются к порту 443, остальные - к 5222.
- Если клиент не находит сервер (например, DNS в сети не отвечает на SRV-запросы), укажите вручную в расширенных настройках учетной записи сервер `xmpp.inkov.dev` и порт `5222`. Порт 443 при ручной настройке указывайте, только если в клиенте есть параметр прямого TLS (Direct TLS): на порту 443 нет STARTTLS.

### 10.4. Проверка функций

1. Войдите как `admin@inkov.dev` на телефоне и как `test@inkov.dev` на компьютере.
2. Добавьте друг друга в контакты и обменяйтесь сообщениями.
3. Проверьте шифрование OMEMO: в Conversations оно включено по умолчанию, в Gajim и Dino включается в окне чата.
4. Отправьте фотографию: ссылка на файл начинается с `https://upload.inkov.dev/file_share/`, без номера порта.
5. Создайте групповой чат: его адрес будет вида `room@conference.inkov.dev`.
6. Позвоните между устройствами в разных сетях, например через мобильный интернет и Wi-Fi: так проверяется TURN.
7. Добавьте контакт с другого XMPP-сервера и обменяйтесь сообщениями: так проверяется федерация.
8. Удалите тестового пользователя: `sudo prosodyctl deluser test@inkov.dev`.

---

## 11. Безопасность и обслуживание

### 11.1. Доступ

- Вход на сервер только по SSH-ключу, вход под root запрещен (раздел 2.4). Веб-консоль Cockpit отключена и закрыта в файрволе (разделы 2.8 и 5).
- SELinux работает в режиме enforcing. Не отключайте его для устранения ошибок: причину отказа показывает `sudo ausearch -m avc -ts recent`.
- Регистрация закрыта (`allow_registration = false`), учетные записи создает только администратор.
- Для XMPP и sudo используйте разные длинные пароли.
- Порт 5280 и консоль `prosodyctl shell` доступны только локально, наружу их не открывайте.
- Токен Cloudflare дает право менять DNS зоны `inkov.dev`. При подозрении на утечку удалите его в **My Profile → API Tokens** и создайте новый, затем обновите `/root/.secrets/cloudflare.ini`.

### 11.2. Обновления

- Обновления безопасности устанавливает dnf5-automatic (раздел 2.9).
- Раз в месяц устанавливайте остальные обновления и проверяйте, нужна ли перезагрузка:

```bash
sudo dnf upgrade --refresh
sudo dnf needs-restarting
```

- Prosody обновляется из репозитория Fedora вместе с системой, в пределах одного выпуска Fedora - в ветке 13.x. Смена основной версии Prosody возможна при переходе на следующий выпуск Fedora (раздел 11.6). Чтобы обновлять Prosody только вручную: `sudo dnf versionlock add prosody`, снять блокировку: `sudo dnf versionlock delete prosody`.
- При обновлении пакетов `prosody` и `coturn` dnf перезапускает их службы в конце транзакции, в том числе при автоматической установке обновлений безопасности. Клиенты переподключаются сами, текущие звонки через TURN прерываются. Службы, которые работают со старыми версиями библиотек, показывает `sudo dnf needs-restarting -s`.

### 11.3. Резервное копирование

Файл базы нельзя копировать через `cp` или `tar` при работающем Prosody: копия может оказаться несогласованной. Утилита `sqlite3` снимает согласованную копию без остановки сервера.

```bash
# Утилита sqlite3 (один раз)
sudo dnf install sqlite
# Согласованная копия базы на работающем сервере
sudo sqlite3 /var/lib/prosody/prosody.sqlite ".backup /root/prosody-$(date +%F).sqlite"
# Проверка копии: ожидается ok
sudo sqlite3 /root/prosody-$(date +%F).sqlite "PRAGMA integrity_check;"
# Архив: копия базы, конфигурация, сертификаты, секреты и drop-in службы.
# Загруженные пользователями файлы не включаются. --ignore-failed-read пропускает
# отсутствующие пути (coturn и модули из раздела 14 необязательны)
sudo tar --ignore-failed-read -czf /root/xmpp-backup-$(date +%F).tar.gz \
  /root/prosody-$(date +%F).sqlite /etc/prosody /etc/pki/prosody /etc/letsencrypt \
  /etc/systemd/system/prosody.service.d /etc/coturn /etc/pki/coturn \
  /usr/local/lib/prosody /root/.secrets
sudo ls -lh /root/xmpp-backup-*.tar.gz
```

Сообщения tar `Removing leading ... from member names` и `Warning: Cannot stat: No such file or directory` для необязательных путей, которых нет на сервере, ожидаемы. Копирование архива на локальный компьютер:

```bash
# На VPS: копия архива в домашний каталог пользователя с правами только для него
sudo install -m 600 -o "$USER" /root/xmpp-backup-$(date +%F).tar.gz ~/
# На локальном компьютере
scp <USER>@xmpp.inkov.dev:~/xmpp-backup-*.tar.gz .
# На VPS: удаление копии из домашнего каталога
rm ~/xmpp-backup-*.tar.gz
```

Восстановление на сервере, подготовленном по разделам 2-8:

```bash
# Файлы конфигурации, сертификаты, секреты и копия базы в /root
sudo tar -xzf xmpp-backup-<дата>.tar.gz -C /
# Метки SELinux для восстановленных файлов
sudo restorecon -R /etc/prosody /etc/pki/prosody /etc/letsencrypt /etc/coturn /etc/pki/coturn /etc/systemd/system/prosody.service.d /root/.secrets
[ -d /usr/local/lib/prosody ] && sudo restorecon -R /usr/local/lib/prosody
sudo systemctl daemon-reload
# База заменяется при остановленном Prosody
sudo systemctl stop prosody
sudo install -o prosody -g prosody -m 640 /root/prosody-<дата>.sqlite /var/lib/prosody/prosody.sqlite
sudo restorecon /var/lib/prosody/prosody.sqlite
sudo systemctl start prosody
sudo systemctl restart coturn
sudo prosodyctl shell user list inkov.dev
```

Булев параметр SELinux в архив не входит: на новом сервере выполните `sudo setsebool -P prosody_bind_http_port on` (раздел 4.3).

> **Важно:** архив содержит базу с учетными записями и перепиской, закрытый ключ сертификата, токен Cloudflare и секрет TURN. Храните копию в зашифрованном виде.

### 11.4. Журналы

- Prosody: `/var/log/prosody/prosody.log` и `prosody.err`. Пакет настраивает ротацию через logrotate.
- coturn: `/var/log/coturn/turnserver.log`.
- certbot: `/var/log/letsencrypt/letsencrypt.log`.
- SSH: `journalctl -u sshd`.
- SELinux: `sudo ausearch -m avc -ts today`. Если команда не найдена: `sudo dnf install audit`.

Отказы SELinux с `name_connect` для `prosody_t` к портам других серверов возможны и ожидаемы. Prosody 13 сначала пробует подключиться к другому серверу по прямому TLS, если у того есть запись `_xmpps-server`. Политика разрешает Prosody исходящие подключения только к портам XMPP, поэтому после отказа Prosody подключается через 5269.

### 11.5. fail2ban

Опционально. Вход по SSH возможен только по ключу, поэтому подбор пароля SSH невозможен. Prosody 13 по умолчанию не записывает в журнал IP-адреса неудачных попыток входа, поэтому для fail2ban нужен сторонний модуль `mod_log_auth` (https://modules.prosody.im/mod_log_auth). При длинных случайных паролях подбор перебором неэффективен, поэтому для личного сервера fail2ban необязателен.

### 11.6. Срок поддержки и переход на следующий выпуск Fedora

Fedora 44 вышла 28 апреля 2026 года. Поддержка заканчивается через 4 недели после выхода Fedora 46, по текущему плану - 2 июня 2027 года. После этой даты обновления безопасности не выпускаются. Переходите на следующий выпуск в течение нескольких месяцев после его выхода, не дожидаясь окончания поддержки.

1. Сделайте резервную копию (11.3).
2. Проверьте, какая версия Prosody будет в новом выпуске. Если основная версия меняется (например, 13 на 14), прочитайте заметки к выпуску на https://prosody.im. При включенном разделе 14 проверьте совместимость модулей (14.8).
3. Выполните обновление:

```bash
# Установка всех обновлений текущего выпуска
sudo dnf upgrade --refresh
# Версия Prosody в следующем выпуске (45 - номер нового выпуска)
dnf repoquery --releasever=45 --latest-limit 1 prosody coturn certbot
# Загрузка пакетов нового выпуска
sudo dnf system-upgrade download --releasever=45
# Перезагрузка и установка. Сервер недоступен несколько минут
sudo dnf system-upgrade reboot
```

4. Проверьте систему и службы:

```bash
cat /etc/fedora-release
getsebool prosody_bind_http_port
sudo prosodyctl check
systemctl status prosody coturn --no-pager
sudo certbot renew --dry-run
```

Булев параметр SELinux, drop-in службы, правила firewalld и переопределение зеркала из раздела 2.2 сохраняются при обновлении. Порядок обновления описан в документации Fedora (ссылка в разделе 13).

---

## 12. Устранение неполадок

| Симптом | Вероятная причина | Как проверить | Как исправить |
|---|---|---|---|
| В журнале Prosody ошибка привязки к порту 443 с `Permission denied` | Нет drop-in с `AmbientCapabilities` или выключен `prosody_bind_http_port` | `systemctl show prosody -p AmbientCapabilities`, `getsebool prosody_bind_http_port`, `sudo ausearch -m avc -ts recent` | Повторить раздел 4.3, затем `sudo systemctl restart prosody` |
| Prosody запущен вручную (`prosody` или `prosodyctl start`), порт 443 не открывается | Право на порт дает только служба systemd | `systemctl status prosody --no-pager` | Остановить ручной запуск, запускать `sudo systemctl start prosody` |
| В `prosody.err` `LuaDBI or LuaSQLite3 are required for using SQL databases`, `prosodyctl adduser` сообщает `no data storage active`, вход не работает | Не установлен `lua-dbi` | `lua -e 'require "DBI"'` | `sudo dnf install lua-dbi`, затем `sudo systemctl restart prosody` |
| `prosody.err` быстро растет, в нем длинные трассировки `stack traceback` | Чаще всего хранилище не работает (строка выше) | `sudo grep -m 5 -v -E '^\s' /var/log/prosody/prosody.err` | Устранить первую ошибку перед трассировкой |
| Prosody не запускается после правки конфигурации | Синтаксическая ошибка Lua: пропущена кавычка, скобка или `;` | `sudo prosodyctl check config` показывает строку с ошибкой | Исправить строку или вернуть `/etc/prosody/prosody.cfg.lua.orig` |
| В журнале `Address already in use` | Порт занят другим процессом | `sudo ss -tulpn`, найти порт в выводе | Остановить процесс (например, предустановленный веб-сервер на 443) |
| `dnf makecache`: `Curl error` или `Failed to download metadata` для `fedora` и `updates` | Сервис выбора зеркал или зеркала недоступны из сети хостера | `curl -sI 'https://mirrors.fedoraproject.org/metalink?repo=fedora-44&arch=x86_64'` | Зеркало Яндекса (раздел 2.2) |
| При первом выпуске сертификата certbot сообщает `reported error code 1` и `No certificates imported :(` | В конфигурации Prosody по умолчанию нет хоста `inkov.dev` | - | Ничего не делать: импорт выполняется в разделе 9.2 |
| Сертификат не продлевается, `systemctl list-timers` не показывает `certbot-renew.timer` | Таймер не включен после установки certbot | `systemctl is-enabled certbot-renew.timer` | `sudo systemctl enable --now certbot-renew.timer` |
| После продления сертификат в Prosody старый | В `/etc/sysconfig/certbot` задан `DEPLOY_HOOK`, он заменяет команду импорта | `grep HOOK /etc/sysconfig/certbot` | Очистить `DEPLOY_HOOK`, выполнить `sudo prosodyctl --root cert import /etc/letsencrypt/live` |
| Клиент не находит сервер | SRV-записи не созданы или еще не видны резолверу; закрыты порты 443 и 5222 | `dig +short SRV _xmpps-client._tcp.inkov.dev`, `nc -vz xmpp.inkov.dev 443` | Проверить записи (раздел 3), firewalld (`sudo firewall-cmd --list-services`) и файрвол хостера |
| Клиент подключается через 5222, но не через 443 | Нет записи `_xmpps-client`, клиент не поддерживает XEP-0368 или закрыт порт 443 | Проверки из 9.4 | Создать запись, открыть порт. Клиенты без XEP-0368 подключаются к 5222 |
| Клиент сообщает об ошибке сертификата | В сертификате нет `inkov.dev`, сертификат не импортирован или истек | Команды `openssl` из 9.4, `sudo prosodyctl check certs` | Выпустить сертификат с `-d inkov.dev -d '*.inkov.dev'`, выполнить `sudo prosodyctl --root cert import /etc/letsencrypt/live` |
| certbot: `Error determining zone_id: 9109 ... Did you enter a valid Cloudflare Token?` | Неверный токен | Проверка токена из 6.3 | Скопировать токен заново или создать новый |
| certbot: `Unable to determine zone_id for inkov.dev` | У токена нет доступа к зоне | Настройки токена в Cloudflare | Zone Resources: `Include` / `Specific zone` / `inkov.dev` |
| certbot: `Error communicating with the Cloudflare API` с подсказкой про `Zone:DNS:Edit` | У токена нет права редактировать DNS | Permissions токена | `Zone` / `DNS` / `Edit` |
| certbot или проверка токена завершается по таймауту | API Cloudflare недоступен из сети хостера | Проверка токена из 6.3 | Ручная проверка DNS-01 (6.8, вариант 1) |
| certbot: `DNS problem: NXDOMAIN looking up TXT` или `Incorrect TXT record` | TXT-запись не успела распространиться | Повторить команду | Увеличить `--dns-cloudflare-propagation-seconds` до 120 |
| Сообщения на другие серверы не доходят | Закрыт порт 5269, нет SRV `_xmpp-server`, у удаленного сервера недействительный сертификат | `nc -vz xmpp.inkov.dev 5269`, `sudo tail -n 50 /var/log/prosody/prosody.log` | Открыть порт, проверить SRV; для отдельного домена с недействительным сертификатом добавить `s2s_insecure_domains = { "example.org" }` в глобальную часть конфигурации |
| В журнале аудита отказы `name_connect` для `prosody_t` | Попытка исходящего подключения по прямому TLS к серверу с записью `_xmpps-server` | `sudo ausearch -m avc -ts today` | Ничего не делать: Prosody подключается к этим серверам через 5269 (раздел 11.4) |
| Не отправляются файлы | Закрыт порт 443, нет записи `upload`, превышен лимит | `nc -vz xmpp.inkov.dev 443`, `curl` из 9.4 | Открыть порт, создать запись, изменить лимиты в компоненте `upload.inkov.dev` |
| Ссылки на файлы содержат `:5280` или `http://` | В компоненте `upload.inkov.dev` нет `http_external_url` | Строка `Serving 'file_share' at` в журнале Prosody | Добавить `http_external_url = "https://upload.inkov.dev/"` в компонент и перезапустить Prosody |
| Звонки не соединяются или нет звука | coturn остановлен, закрыт диапазон 50000-50100/udp, секреты не совпадают, VPS за NAT без `external-ip` | `systemctl status coturn`, `sudo prosodyctl check turn -v --ping=stun.conversations.im` | Запустить coturn, открыть порты, повторить подстановку секрета, указать `external-ip` |
| `check turn`: `STUN returned a private IP` | VPS за NAT | `ip -br addr` | `external-ip=<IP_VPS>/<ВНУТРЕННИЙ_IP>` в `/etc/coturn/turnserver.conf`, затем `sudo systemctl restart coturn` |
| coturn останавливается после `systemctl reload coturn` или ротации журнала | В конфигурации `syslog` вместо `log-file`: coturn 4.18 завершается по сигналу HUP | `grep -e '^syslog' -e '^log-file' /etc/coturn/turnserver.conf` | Вернуть `log-file` и `simple-log` (раздел 8.2), `sudo systemctl restart coturn` |
| В журнале coturn `Bad configuration format: no-dtls` или `no-cli option is deprecated` | Параметры из конфигурации для старых версий coturn | `sudo grep -e WARNING -e ERROR /var/log/coturn/turnserver.log` | Удалить `no-dtls`, `no-cli`, `no-software-attribute`, `no-tlsv1`, `no-tlsv1_1` |
| Prosody: `Permission denied` при загрузке ключа | В конфигурации указан путь к ключу в `/etc/letsencrypt` | `sudo cat /var/log/prosody/prosody.err` | Удалить явные пути к сертификатам, выполнить `cert import` |
| После раздела 2.4 не удается войти по SSH | Ключ не скопирован, вход проверялся под другим пользователем или у `~/.ssh` неверная метка SELinux | Консоль хостера: `ls -lZ /home/<USER>/.ssh/authorized_keys` | `sudo restorecon -R /home/<USER>/.ssh`; при необходимости `sudo rm /etc/ssh/sshd_config.d/10-hardening.conf && sudo systemctl restart sshd` и повторить раздел 2.3 |
| После изменения правил firewalld пропал доступ по SSH | Удалена служба `ssh` или SSH работает на другом порту | Консоль хостера: `sudo firewall-cmd --list-all` | `sudo firewall-cmd --permanent --add-service=ssh` (или `--add-port=<PORT>/tcp`), `sudo firewall-cmd --reload` |
| `check dns`: `No _xmpps-server SRV record found ..., but it looks like you need one.` | Особенность проверки при `net_multiplex` | Раздел 9.3 | Ничего не делать: связь с другими серверами работает через 5269 |
| `check dns`: `inkov.dev A record points to unknown address 185.199.x.x` | SRV-записи не видны резолверу сервера | `dig +short SRV _xmpp-client._tcp.inkov.dev` на VPS | Проверить SRV-записи в Cloudflare, подождать до 30 минут |

Подробный журнал Prosody: в `/etc/prosody/prosody.cfg.lua` замените `info =` на `debug =` в блоке `log`, выполните `sudo systemctl restart prosody`, воспроизведите проблему и верните `info`.

---

## 13. Резюме

1. Prosody 13 из репозитория Fedora на VPS с Fedora Server 44 обслуживает адреса `user@inkov.dev`, сайт `inkov.dev` остается на GitHub Pages. Данные хранятся в базе SQLite `/var/lib/prosody/prosody.sqlite`.
2. Клиенты подключаются через порт 443 (прямой TLS) или 5222, другие серверы - через 5269. SRV-записи в зоне `inkov.dev` указывают на `xmpp.inkov.dev`.
3. Порт 443 открыт для Prosody через `AmbientCapabilities` в drop-in systemd и булев параметр SELinux `prosody_bind_http_port`. SELinux остается в режиме enforcing.
4. Обмен файлами работает по HTTPS на порту 443, ссылки на файлы без номера порта. Звонки идут через coturn.
5. Сертификат на `inkov.dev` и `*.inkov.dev` выпускается через DNS-01 и API Cloudflare, продлевается по таймеру `certbot-renew.timer` и импортируется в Prosody.
6. Вход на сервер только по SSH-ключу, Cockpit отключен, firewalld открывает только нужные порты, регистрация на XMPP-сервере закрыта, coturn не пересылает трафик на внутренние адреса.
7. Раз в 6-12 месяцев сервер обновляется до следующего выпуска Fedora.

### Чек-лист

- [ ] ОС обновлена, SELinux в режиме enforcing, пользователь из группы `wheel` входит по ключу, вход по паролю и под root отключен (раздел 2)
- [ ] Cockpit отключен, dnf5-automatic устанавливает обновления безопасности (разделы 2.8 и 2.9)
- [ ] DNS-записи созданы в режиме DNS only и видны через `dig` (раздел 3)
- [ ] Prosody и `lua-dbi` установлены, `prosodyctl about` показывает Lua 5.4 и `LuaDBI` (раздел 4.2)
- [ ] Drop-in с `AmbientCapabilities` создан, `prosody_bind_http_port` включен (раздел 4.3)
- [ ] firewalld пропускает `https`, `xmpp-client`, `xmpp-server`, служба `cockpit` удалена из зоны (раздел 5)
- [ ] Сертификат выпущен, `certbot-renew.timer` включен, `sudo certbot renew --dry-run` проходит (раздел 6)
- [ ] `sudo prosodyctl check config` без ошибок (раздел 7)
- [ ] `sudo prosodyctl check turn -v` показывает `Success!` (раздел 8)
- [ ] После запуска создан `/var/lib/prosody/prosody.sqlite`, в `prosody.err` нет ошибок хранилища (раздел 9.1)
- [ ] Сертификат импортирован, порт 443 отвечает для XMPP и HTTPS, `check certs` и `check connectivity` без ошибок (раздел 9)
- [ ] Сообщения, файлы, групповые чаты, звонки и федерация работают (раздел 10)
- [ ] Резервная копия создана и перенесена с сервера (раздел 11)
- [ ] Дата перехода на следующий выпуск Fedora запланирована (раздел 11.6)

### Меры безопасности

Сервер доступен из интернета, поэтому его безопасность зависит от регулярного обслуживания. Раз в месяц обновляйте пакеты и проверяйте срок сертификата (`sudo certbot certificates`). Не отключайте SELinux, не открывайте наружу порты 5280 и 9090, не включайте вход по паролю в SSH и не отключайте `s2s_secure_auth` и `c2s_require_encryption`. Храните SSH-ключ, файлы из `/root/.secrets` и резервные копии в защищенном месте. Переходите на следующий выпуск Fedora до окончания поддержки текущего.

### Документация

- Prosody: https://prosody.im/doc
- Установка Prosody в Fedora: https://prosody.im/download/start
- Конфигурация: https://prosody.im/doc/configure
- Мультиплексирование портов: https://prosody.im/doc/modules/mod_net_multiplex
- Сертификаты: https://prosody.im/doc/certificates, Let's Encrypt: https://prosody.im/doc/letsencrypt
- DNS: https://prosody.im/doc/dns
- TURN: https://prosody.im/doc/turn
- mod_http_file_share: https://prosody.im/doc/modules/mod_http_file_share
- mod_turn_external: https://prosody.im/doc/modules/mod_turn_external
- mod_mam: https://prosody.im/doc/modules/mod_mam
- mod_muc: https://prosody.im/doc/modules/mod_muc
- prosodyctl: https://prosody.im/doc/prosodyctl
- Хранилища данных: https://prosody.im/doc/storage
- mod_storage_sql: https://prosody.im/doc/modules/mod_storage_sql
- Сторонние модули: https://prosody.im/doc/installing_modules
- XEP-0368 (прямой TLS): https://xmpp.org/extensions/xep-0368.html
- Fedora Server: https://docs.fedoraproject.org/en-US/fedora-server/
- Жизненный цикл выпусков Fedora: https://docs.fedoraproject.org/en-US/releases/lifecycle/
- Обновление Fedora до следующего выпуска: https://docs.fedoraproject.org/en-US/quick-docs/upgrading-fedora-offline/
- SELinux в Fedora: https://docs.fedoraproject.org/en-US/quick-docs/selinux-getting-started/
- firewalld: https://docs.fedoraproject.org/en-US/quick-docs/firewalld/, https://firewalld.org/documentation/
- coturn: https://github.com/coturn/coturn
- certbot: https://eff-certbot.readthedocs.io/en/stable/
- Плагин certbot для Cloudflare: https://certbot-dns-cloudflare.readthedocs.io/en/stable/
- DNS в Cloudflare: https://developers.cloudflare.com/dns/
- API-токены Cloudflare: https://developers.cloudflare.com/fundamentals/api/get-started/create-token/

---

## 14. Быстрое подключение: SASL2, Bind 2, FAST (опционально)

Раздел выполняется после раздела 10, когда сервер уже работает. Он ускоряет подключение и переподключение мобильных клиентов. Для работы сервера раздел не нужен.

### 14.1. Что это дает

- **XEP-0388 (SASL2)** - новый протокол входа вместо SASL из RFC 6120. Клиент передает свой идентификатор, название программы и устройство. После успешного входа поток не перезапускается, как при обычном SASL, а в сам запрос входа можно встроить дополнительные действия.
- **XEP-0386 (Bind 2)** - привязка ресурса внутри запроса SASL2. В этом же запросе клиент включает Carbons и сообщает состояние CSI. Модуль `mod_sasl2_sm` добавляет в запрос включение и возобновление Stream Management (XEP-0198).
- **XEP-0484 (FAST)** - вход по токену. После первого входа по паролю клиент получает токен и дальше входит по нему. Токен действует 21 день и заменяется новым раз в сутки. Смена пароля делает недействительными все выданные токены.

XEP-0388 и XEP-0386 имеют статус Stable. В ядре Prosody 13 их нет: они реализованы в сторонних модулях из репозитория prosody-modules со статусом Beta.

Количество обменов с сервером при переподключении с возобновлением сессии, после установки TCP-соединения:

| Этап | Обычный SASL | SASL2, Bind 2, SM | SASL2, Bind 2, SM, FAST |
|---|---|---|---|
| TLS и открытие потока | 2 | 2 | 2 |
| Вход | 2 (SCRAM) | 2 (SCRAM) | 1 (токен) |
| Перезапуск потока | 1 | - | - |
| Возобновление сессии | 1 | внутри входа | внутри входа |
| Всего | около 6 | около 4 | около 3 |

В мобильной сети с задержкой 150-300 мс переподключение сокращается примерно на полсекунды-секунду. Это заметно при частой смене сети и на iOS: после push-уведомления у приложения мало времени, чтобы подключиться и забрать сообщения.

Что не меняется: федерация, обмен файлами, звонки, OMEMO. Клиенты без поддержки SASL2 продолжают входить по обычному SASL: сервер предлагает оба варианта.

Поддержка в клиентах из раздела 10.2:

| Клиент | SASL2, Bind 2, FAST |
|---|---|
| Conversations, Cheogram, Monocles Chat | да |
| Monal | да |
| Gajim | SASL2 есть, поддержка Bind 2 и FAST не подтверждена |
| Dino | в разработке, в стабильном выпуске не подтверждено |

### 14.2. Риски и когда включать

Риски:

1. **Сторонние модули со статусом Beta.** Их нет в пакете Fedora, `dnf upgrade` их не обновляет. Руководство проверено на ревизии `9503fcbf014f` репозитория prosody-modules с Prosody 13.0.6. Более новые ревизии ориентированы на следующую версию Prosody и могут использовать функции, которых нет в 13.x.
2. **Смена основной версии Prosody.** При переходе на следующий выпуск Fedora Prosody может обновиться до версии 14. Модули могут перестать загружаться или оказаться в ядре Prosody и конфликтовать с копиями в `/usr/local/lib/prosody/modules`. Если модуль не загрузится, Prosody запишет ошибку, и клиенты вернутся к обычному SASL. Если модуль загрузится, но будет работать неправильно, перестанет работать вход у клиентов с поддержкой SASL2, то есть у основных мобильных клиентов. Откат занимает минуту (14.9).
3. **Токены FAST.** Токен на устройстве дает доступ к учетной записи без пароля до 21 дня. Отозвать все токены пользователя можно сменой пароля (`sudo prosodyctl passwd`), токены отдельного устройства - командой из 14.6.
4. **Диагностика.** `prosodyctl check` эти модули не проверяет. Ошибки входа разбираются по журналу в режиме `debug`.

Модули ставятся копированием файлов, а не командой `prosodyctl install`. Установщик Prosody использует luarocks, а пакет `luarocks` в Fedora требует компилятор gcc, `zip` и `unzip` и по умолчанию ставит заголовки Lua 5.1 (`compat-lua-devel`). Серверу эти пакеты не нужны.

Когда включать:

- Несколько пользователей, стабильная связь, в основном настольные клиенты: выигрыш почти незаметен, `smacks` уже восстанавливает сессию без потери сообщений.
- Основные клиенты Conversations и Monal, частая смена сети, iOS: польза есть.
- Нужен список устройств, у которых есть доступ к учетной записи: включите вместе с `mod_client_management` (14.6).

### 14.3. Установка модулей

Модули копируются в отдельный каталог `/usr/local/lib/prosody/modules`. Файлы в нем получают метку SELinux `lib_t`, которую Prosody может читать. Файл `.hg_archival.txt` из архива хранит ревизию, ее показывает `prosodyctl about`.

```bash
# Ревизия prosody-modules, на которой проверено руководство
REV=9503fcbf014f
# Модули: SASL2, Bind 2, Stream Management внутри входа, FAST
MODS="mod_sasl2 mod_sasl2_bind2 mod_sasl2_sm mod_sasl2_fast"
tmp=$(mktemp -d)
curl -fsSL "https://hg.prosody.im/prosody-modules/archive/$REV.tar.gz" | tar -xz -C "$tmp" --strip-components=1
# Совместимость по README: ожидается строка "13 ... Works" для каждого модуля
grep -H -E '^ *13 ' $(for m in $MODS; do echo "$tmp/$m/README.md"; done)
# Каталог для сторонних модулей и копирование
sudo install -d -m 755 /usr/local/lib/prosody/modules
for m in $MODS; do sudo cp -r "$tmp/$m" /usr/local/lib/prosody/modules/; done
sudo cp "$tmp/.hg_archival.txt" /usr/local/lib/prosody/modules/
rm -rf "$tmp"
# Метки SELinux по умолчанию и проверка
sudo restorecon -R /usr/local/lib/prosody
ls -Z /usr/local/lib/prosody/modules
```

**Результат:** для каждого модуля строка с `13` и `Works` (в README `mod_sasl2_sm` и `mod_sasl2_fast` написано `Work`), в каталоге четыре подкаталога `mod_sasl2*` с меткой `lib_t`, например `unconfined_u:object_r:lib_t:s0 mod_sasl2`.

- Файлы копируются командой `cp`, а не `mv`: при перемещении из `/tmp` файлы сохранили бы метку временного каталога, и Prosody не смог бы их прочитать.
- Если `hg.prosody.im` недоступен с VPS, скачайте архив `https://hg.prosody.im/prosody-modules/archive/9503fcbf014f.tar.gz` на другом компьютере, перенесите на VPS через `scp` и выполните команды, начиная с `tar -xz`, указав файл: `tar -xzf <архив> -C "$tmp" --strip-components=1`.

### 14.4. Конфигурация

```bash
# Резервная копия и откат: sudo cp /etc/prosody/prosody.cfg.lua.before-sasl2 /etc/prosody/prosody.cfg.lua
sudo cp /etc/prosody/prosody.cfg.lua /etc/prosody/prosody.cfg.lua.before-sasl2
sudo nano /etc/prosody/prosody.cfg.lua
```

В глобальной части, после строки `certificates = "/etc/pki/prosody"`, добавьте:

```lua
-- Сторонние модули (раздел 14): SASL2, Bind 2, FAST
plugin_paths = { "/usr/local/lib/prosody/modules" }
```

В `modules_enabled`, после строки `"net_multiplex";`, добавьте:

```lua
    -- Быстрое подключение (раздел 14)
        "sasl2"; -- вход по SASL2 (XEP-0388)
        "sasl2_bind2"; -- привязка ресурса внутри входа (XEP-0386)
        "sasl2_sm"; -- возобновление сессии внутри входа (XEP-0198)
        "sasl2_fast"; -- вход по токену (XEP-0484)
```

Срок действия токена FAST можно сократить (необязательно). Строка добавляется в глобальную часть рядом с `plugin_paths`:

```lua
-- Срок действия токена FAST: 7 дней вместо 21. Клиент, не подключавшийся дольше, снова запросит пароль
sasl2_fast_token_ttl = 7 * 86400
```

Проверка и перезапуск:

```bash
sudo prosodyctl check config
sudo systemctl restart prosody
```

**Результат:** `All checks passed, congratulations!`. Новые модули и `plugin_paths` загружаются только при запуске, поэтому нужен `restart`, а не `reload`. Клиенты при перезапуске отключатся и подключатся заново.

### 14.5. Проверка

```bash
# Каталог модулей подключен, ревизия определена
sudo prosodyctl about | grep prosody-modules
# Ошибки загрузки модулей: ожидается пустой вывод (-s: файла может не быть)
sudo grep -s -i sasl2 /var/log/prosody/prosody.err
```

**Результат:**

```text
  /usr/local/lib/prosody/modules - prosody-modules rev: 9503fcbf014f
```

Проверка на сервере или на локальном компьютере: Prosody предлагает SASL2, Bind 2, FAST и Stream Management.

```bash
(printf "<?xml version='1.0'?><stream:stream to='inkov.dev' from='admin@inkov.dev' xmlns='jabber:client' xmlns:stream='http://etherx.jabber.org/streams' version='1.0'>"; sleep 3) \
  | timeout 6 openssl s_client -connect xmpp.inkov.dev:443 -servername inkov.dev -alpn xmpp-client -quiet 2>/dev/null \
  | grep -o -E 'urn:xmpp:(sasl:2|bind:0|fast:0|sm:3)' | sort -u
```

**Результат:**

```text
urn:xmpp:bind:0
urn:xmpp:fast:0
urn:xmpp:sasl:2
urn:xmpp:sm:3
```

Атрибут `from` в заголовке потока нужен для проверки FAST: Prosody предлагает FAST, только если клиент указал свой адрес. Клиенты с поддержкой SASL2 указывают его сами. Если строки `urn:xmpp:sasl:2` нет, модули не загружены: проверьте `plugin_paths`, список `modules_enabled` и перезапуск.

После перезапуска Prosody Conversations и Monal переходят на SASL2 при следующем подключении, отдельная настройка в клиенте не нужна. Какой способ входа использует клиент, показывает список устройств (14.6).

### 14.6. Список устройств (опционально)

Модуль `mod_client_management` показывает клиентов, у которых есть доступ к учетной записи, и отзывает доступ. Он использует данные о клиенте из SASL2 и токены FAST, поэтому требует `mod_sasl2_fast`.

```bash
REV=9503fcbf014f
tmp=$(mktemp -d)
curl -fsSL "https://hg.prosody.im/prosody-modules/archive/$REV.tar.gz" | tar -xz -C "$tmp" --strip-components=1
sudo cp -r "$tmp/mod_client_management" /usr/local/lib/prosody/modules/
rm -rf "$tmp"
sudo restorecon -R /usr/local/lib/prosody
sudo nano /etc/prosody/prosody.cfg.lua
```

В `modules_enabled` после строки `"sasl2_fast";` добавьте:

```lua
        "client_management"; -- список клиентов с доступом к учетной записи
```

```bash
sudo prosodyctl check config && sudo systemctl restart prosody
# Клиенты учетной записи
sudo prosodyctl shell user clients admin@inkov.dev
```

**Результат** (пример):

```text
ID                    | Software              |  First seen |  Last seen |    Expires | Authentication
------------------------------------------------------------------------------------------------------
client/d4565fa7-4d72… | Conversations         |    09:15:58 |   09:16:05 |            | connected, fast, password
------------------------------------------------------------------------------------------------------
OK: 1 clients
```

В колонке `Authentication`: `connected` - у клиента есть сессия, `password` - клиент входил по паролю, `fast` - у клиента есть действующий токен FAST. Клиенты без SASL2 в списке определяются только по ресурсу и могут отображаться неточно.

Отзыв доступа устройства по имени программы из колонки `Software`:

```bash
sudo prosodyctl shell user revoke_client admin@inkov.dev software/Conversations
```

Команда закрывает сессию клиента и отзывает его токены FAST. Если клиент хотя бы раз входил по паролю, команда дополнительно выводит `Error: Password reset required`: пароль по-прежнему сохранен на устройстве, и клиент может войти снова. В этом случае смените пароль командой `sudo prosodyctl passwd admin@inkov.dev`, это отзывает и все токены FAST учетной записи. Так как первый вход всегда выполняется по паролю, для потерянного устройства надежный способ - смена пароля. Основная польза модуля - список устройств.

### 14.7. Смена пароля и токены

Токен FAST, выданный до смены пароля, Prosody отклоняет с ошибкой `credentials-expired`. После `sudo prosodyctl passwd` все устройства пользователя запросят новый пароль при следующем подключении. Это поведение проверено на Prosody 13.0.6 и ревизии `9503fcbf014f`.

### 14.8. Обновление модулей

Обновляйте модули при переходе на новую версию Prosody или при исправлениях в модулях. Перед обновлением проверьте таблицу совместимости в README каждого модуля.

```bash
# Новая ревизия: номер из https://hg.prosody.im/prosody-modules/ или tip для последней
REV=<НОВАЯ_РЕВИЗИЯ>
MODS="mod_sasl2 mod_sasl2_bind2 mod_sasl2_sm mod_sasl2_fast"
# Добавьте mod_client_management, если он установлен (14.6)
tmp=$(mktemp -d)
curl -fsSL "https://hg.prosody.im/prosody-modules/archive/$REV.tar.gz" | tar -xz -C "$tmp" --strip-components=1
# Совместимость с установленной версией Prosody
grep -H -A 6 -i 'compatibility' $(for m in $MODS; do echo "$tmp/$m/README.md"; done)
# Копия текущих модулей для отката
sudo cp -a /usr/local/lib/prosody/modules /usr/local/lib/prosody/modules.bak-$(date +%F)
# Замена модулей целиком: удаленные в новой ревизии файлы не остаются
for m in $MODS; do
  sudo rm -rf "/usr/local/lib/prosody/modules/$m"
  sudo cp -r "$tmp/$m" /usr/local/lib/prosody/modules/
done
sudo cp "$tmp/.hg_archival.txt" /usr/local/lib/prosody/modules/
rm -rf "$tmp"
sudo restorecon -R /usr/local/lib/prosody
sudo prosodyctl check config && sudo systemctl restart prosody
```

Затем повторите проверки из 14.5. Откат к предыдущей ревизии:

```bash
sudo rm -rf /usr/local/lib/prosody/modules
sudo cp -a /usr/local/lib/prosody/modules.bak-<дата> /usr/local/lib/prosody/modules
sudo systemctl restart prosody
```

### 14.9. Откат

```bash
# Конфигурация до раздела 14
sudo cp /etc/prosody/prosody.cfg.lua.before-sasl2 /etc/prosody/prosody.cfg.lua
sudo prosodyctl check config && sudo systemctl restart prosody
# Файлы модулей (необязательно)
sudo rm -rf /usr/local/lib/prosody
```

Если после раздела 14 в конфигурацию вносились другие изменения, не возвращайте резервную копию, а удалите строку `plugin_paths`, строки `sasl2*` и `client_management` через `sudo nano`. Токены FAST остаются в базе и не используются. Клиенты вернутся к обычному SASL при следующем подключении. Если клиент после отката не подключается, войдите в учетную запись в клиенте заново.

### 14.10. Устранение неполадок

| Симптом | Вероятная причина | Как проверить | Как исправить |
|---|---|---|---|
| В `prosody.err` ошибка загрузки модуля `sasl2` (`module not found`) | Нет `plugin_paths`, файлы в другом каталоге или неверная метка SELinux | `sudo prosodyctl about`, `ls -Z /usr/local/lib/prosody/modules` | Проверить `plugin_paths`, выполнить `sudo restorecon -R /usr/local/lib/prosody` |
| В признаках потока нет `urn:xmpp:sasl:2` | Модули не добавлены в `modules_enabled` или Prosody не перезапущен | Проверка из 14.5 | Добавить модули (14.4), `sudo systemctl restart prosody` |
| Нет `urn:xmpp:fast:0`, остальные строки есть | Не загружен `sasl2_fast` или в заголовке потока нет `from` | Проверка из 14.5 с атрибутом `from` | Добавить `"sasl2_fast";`, перезапустить Prosody |
| После включения модулей Conversations или Monal не входят, другие клиенты входят | Ошибка в модулях или несовместимая ревизия | `debug` в блоке `log`, записи `sasl2` в `prosody.log` | Откат (14.9) или предыдущая ревизия (14.8) |
| `revoke_client` выводит `Error: Password reset required` | Клиент входил по паролю | `sudo prosodyctl shell user clients user@inkov.dev` | Сменить пароль: `sudo prosodyctl passwd user@inkov.dev` |

Документация:

- mod_sasl2: https://modules.prosody.im/mod_sasl2
- mod_sasl2_bind2: https://modules.prosody.im/mod_sasl2_bind2
- mod_sasl2_sm: https://modules.prosody.im/mod_sasl2_sm
- mod_sasl2_fast: https://modules.prosody.im/mod_sasl2_fast
- mod_client_management: https://modules.prosody.im/mod_client_management
- XEP-0388 (SASL2): https://xmpp.org/extensions/xep-0388.html
- XEP-0386 (Bind 2): https://xmpp.org/extensions/xep-0386.html
- XEP-0484 (FAST): https://xmpp.org/extensions/xep-0484.html
- Статья разработчиков Prosody о FAST: https://blog.prosody.im/fast-auth/
