# ansible-forkop-openwrt-24-10

Ansible-плейбук для настройки нового роутера Xiaomi AX3000T под OpenWrt 24.10.x:
LAN, Wi-Fi, Forkop (бывший Podkop Plus) и SQM — по образцу существующего
роутера `192.168.3.1`. WAN (PPPoE) настраивается отдельно, вручную.

## Что делает

1. **network** — проверяет, что WAN поднят и есть интернет. LAN playbook
   не настраивает: адрес, маску и DHCP задай вручную до запуска.
2. **base** — hostname, NTP, выключение IPv6 на сетевом уровне (LAN/WAN, без глобальных sysctl).
3. **wireless** — 2.4 ГГц (канал 1, HE20) и 5 ГГц (канал 36, HE80), WPA2, SSID
   `xiaomi_XXXX` по MAC-адресу радио.
4. **forkop** — установка штатным `install.sh` (forkop + luci-app-forkop + sing-box),
   запись `/etc/config/forkop` (секции `main`/`russian`, подписки, приоритеты, URLTest),
   запуск и ожидание sing-box и DNS `127.0.0.42`.
5. **sqm** — `sqm-scripts` + `luci-app-sqm`, cake на WAN-интерфейсе
   (`pppoe-wan` при PPPoE, `wan` при DHCP — определяется автоматически).
6. **verify** — итоговый отчёт по проверкам.
7. **finalize** — установка пароля root (последним шагом).

Домены и IP-адреса Forkop (`option domain` / `option ip_cidr`) не переносятся —
их настраивают вручную в LuCI → Forkop.

## Требования

- Роутер уже прошит OpenWrt 24.10.x, LAN настроен (например, `192.168.2.1/24`),
  компьютер в той же подсети, SSH доступен. Прошивка в этот playbook не входит.
- WAN настроен и интернет работает: PPPoE (логин и пароль) настраивается
  вручную до запуска playbook. Без интернета не поставить Forkop и SQM.
- Кабель в LAN-порт: Wi-Fi перезапускается по ходу настройки.
- На контроллере Ansible: `ansible-core >= 2.15`, `sshpass` (или `ansible_connection: paramiko`).

## Быстрый старт

```bash
cp vars/router.example.yml vars/router.yml
$EDITOR vars/router.yml            # IP роутера

ansible-playbook playbooks/site.yml -e @vars/router.yml
```

При запуске playbook спросит (пароли на диск не пишутся):

- **hostname** нового роутера;
- **пароль Wi-Fi** (сразу для 2.4 и 5 ГГц) — он же станет новым паролем root;
- **подписки Forkop**: если не указаны в `vars/router.yml`, сначала количество
  URL, затем каждый URL отдельным скрытым вопросом — один и тот же список
  пишется в секции `main` и `russian`;
- **community-списки Forkop** через запятую (по умолчанию
  `russia_inside,youtube,telegram,github,cloudflare`).

Если ответы уже есть в переменных, вопросы пропускаются. Имена переменных
для `vars/router.yml`:

```yaml
router_hostname: kosmos2
router_password: "пароль Wi-Fi и новый пароль root"
forkop_community_lists_input: "russia_inside,youtube,telegram,github,cloudflare"
forkop_subscriptions_input:
  - "https://..."
  - "https://..."
```

Подписки указываются именно списком, по одному URL на строку. Один и тот же
список записывается в `main` и `russian`. Если `forkop_subscriptions_input`
в файле нет, playbook спросит количество и каждый URL отдельно.

Пустыми их не оставляй: пустое значение тоже перебьёт вопрос.

Текущий пароль root вопроса не имеет: по умолчанию он пустой (свежая прошивка).
Если роутер уже настроен, передай его в `ansible_password`:

```bash
ansible-playbook playbooks/site.yml -e @vars/router.yml -e ansible_password='...'
```

Пример запуска без части вопросов:

```bash
ansible-playbook playbooks/site.yml -e @vars/router.yml \
  -e router_hostname=kosmos2 \
  -e forkop_community_lists_input=russia_inside,youtube
```

Пароль Wi-Fi/root лучше не передавать через `-e` — он останется в истории
команд; для него оставь вопрос playbook.

## Переменные

Основные значения в `vars/router.yml` (в `.gitignore`):

| Переменная | Что это |
|---|---|
| `ansible_host` | текущий LAN-адрес роутера (задан вручную) |
| `sqm_enabled`, `sqm_download_kbit`, `sqm_upload_kbit` | шейпинг |
| `sqm_interface` | необязательно: по умолчанию берётся `l3_device` интерфейса `wan` (`pppoe-wan` или `wan`) |

Пароль Wi-Fi (он же новый пароль root) и подписки Forkop
по умолчанию спрашиваются при запуске и в файлы не пишутся. Текущий пароль
root по умолчанию пустой (свежая прошивка); для уже настроенного роутера
передай его через `-e ansible_password='...'`.
LAN playbook не меняет: адрес, маску и DHCP настрой вручную до запуска.
SSID вычисляется сам: `xiaomi_XXXX`, где `XXXX` — последние 4 hex-символа
MAC-адреса радио (2.4 ГГц и 5 ГГц — свои).

Значения по умолчанию в ролях: `roles/*/defaults/main.yml` (каналы,
qdisc `cake`, NTP-серверы и т.п.).

## Как это работает

Роутер без Python, поэтому весь обмен идёт через `raw` + `uci`:

- настройки применяются сгенерированным shell-скриптом (роль `uci`), который
  сравнивает текущее значение с нужным и коммитит только при изменении —
  повторный запуск ничего не меняет и не перезапускает сервисы;
- `/etc/config/forkop` пишется целиком по контрольной сумме; первая версия
  файла сохраняется в `/etc/config/forkop.orig`;
- dnsmasq на `127.0.0.42` переводит сам Forkop, playbook это только проверяет.

## Снять конфиг с эталонного роутера

```bash
ansible-playbook playbooks/dump.yml -e ansible_password="$REFERENCE_ROOT_PASSWORD"
```

Дамп (`uci export`, `/etc/config/forkop`, список пакетов, версия прошивки)
складывается в `backup/<host>-<дата>/` с правами `0600`. Каталог в `.gitignore`,
внутри пароли и токены подписок.

## Проверки и откат

`verify` показывает отчёт: forkop/sing-box/DNS, dnsmasq, WAN, IPv6, SSID, hostname,
SQM. Если проверка не прошла — playbook останавливается с подсказкой.

Откат:

```bash
ssh root@<LAN-адрес роутера>
cp /etc/config/forkop.orig /etc/config/forkop
/etc/init.d/forkop restart
```

или полный сброс к заводским настройкам OpenWrt: `firstboot && reboot`.

## Повторный запуск

`ansible_host` в `vars/router.yml` должен указывать на **текущий** адрес роутера.
Пароль root передай через `-e ansible_password='...'` — это пароль, заданный
в прошлый раз. Пароль root переустанавливается на каждом запуске.

## Замечания

- WAN playbook не трогает: PPPoE (логин и пароль) настраивается вручную до запуска.
  MAC на WAN не клонируется: `macaddr` в `network.wan` не задан, WAN-MAC аппаратный
  и отличается от LAN-MAC (проверено на `192.168.2.1` и `192.168.3.1`).
- `install.sh` в неинтерактивном режиме ставит sing-box **stable** (как на эталоне)
  и `luci-i18n-forkop-ru`.
- Forkop ставится последней версией из GitHub. Конфиг снят с 1.0.5; при выходе
  новой версии миграции выполнит сам Forkop.
- Если GitHub с роутера недоступен, установка Forkop упадёт с соответствующим
  сообщением — тогда `forkop`/`luci-app-forkop` ставятся вручную, остальное
  playbook сделает при повторном запуске.
