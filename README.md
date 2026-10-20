# ansible-forkop-openwrt-24-10

Ansible-плейбук для настройки нового роутера Xiaomi AX3000T под OpenWrt 24.10.x:
LAN, Wi-Fi, Forkop (бывший Podkop Plus) и SQM — по образцу существующего
роутера `192.168.3.1`. WAN (PPPoE) настраивается отдельно, вручную.

## Что делает

1. **network** — проверяет, что WAN уже поднят и есть интернет (PPPoE playbook не настраивает),
   ставит LAN `192.168.1.1/24`.
2. **base** — hostname, NTP, выключение IPv6 на сетевом уровне (LAN/WAN, без глобальных sysctl).
3. **wireless** — 2.4 ГГц (канал 1, HE20) и 5 ГГц (канал 36, HE80), WPA2.
4. **forkop** — установка штатным `install.sh` (forkop + luci-app-forkop + sing-box),
   запись `/etc/config/forkop` (секции `main`/`russian`, подписки, приоритеты, URLTest),
   запуск и ожидание sing-box и DNS `127.0.0.42`.
5. **sqm** — `sqm-scripts` + `luci-app-sqm`, cake на `pppoe-wan`.
6. **verify** — итоговый отчёт по проверкам.
7. **finalize** — установка пароля root (последним шагом).

Домены и IP-адреса Forkop (`option domain` / `option ip_cidr`) не переносятся —
их настраивают вручную в LuCI → Forkop.

## Требования

- Роутер уже прошит OpenWrt 24.10.x и доступен по SSH (свежая прошивка: `192.168.1.1`,
  root без пароля). Прошивка в этот playbook не входит.
- WAN настроен и интернет работает: PPPoE (логин, пароль, при необходимости MAC)
  настраивается вручную до запуска playbook. Без интернета не поставить Forkop и SQM.
- Кабель в LAN-порт: Wi-Fi перезапускается по ходу настройки.
- На контроллере Ansible: `ansible-core >= 2.15`, `sshpass` (или `ansible_connection: paramiko`).

## Быстрый старт

```bash
cp vars/router.example.yml vars/router.yml
$EDITOR vars/router.yml            # пароли, SSID, подписки

ansible-playbook playbooks/site.yml -e @vars/router.yml
```

При запуске playbook спросит:

- **hostname** нового роутера;
- **community-списки Forkop** через запятую (по умолчанию
  `russia_inside,youtube,telegram,github,cloudflare`).

Если ответы уже есть в переменных, вопросы пропускаются:

```bash
ansible-playbook playbooks/site.yml -e @vars/router.yml \
  -e router_hostname=kosmos2 \
  -e forkop_community_lists_input=russia_inside,youtube
```

## Переменные

Все уникальные значения — в `vars/router.yml` (в `.gitignore`, там секреты).

| Переменная | Что это |
|---|---|
| `ansible_host`, `ansible_password` | IP роутера и **текущий** пароль root (пустой на свежей прошивке) |
| `new_root_password` | пароль root, который будет установлен в конце |
| `wifi_2g_ssid` / `wifi_2g_key` | Wi-Fi 2.4 ГГц |
| `wifi_5g_ssid` / `wifi_5g_key` | Wi-Fi 5 ГГц |
| `forkop_main_subscriptions` | список URL подписок для секции `main` |
| `forkop_russian_subscription` | URL подписки для секции `russian` |
| `sqm_enabled`, `sqm_download_kbit`, `sqm_upload_kbit` | шейпинг |

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
ssh root@192.168.1.1
cp /etc/config/forkop.orig /etc/config/forkop
/etc/init.d/forkop restart
```

или полный сброс к заводским настройкам OpenWrt: `firstboot && reboot`.

## Повторный запуск

В `ansible_password` указывай уже **новый** пароль (тот, что в `new_root_password`).
Пароль root переустанавливается на каждом запуске.

## Замечания

- WAN playbook не трогает. На эталоне на WAN руками прописан MAC
  (`cc:d8:43:19:8f:67`), не совпадающий с заводским (`cc:d8:43:b6:3d:9c`) — когда-то его
  клонировали, возможно под провайдера. Если PPPoE на новом роутере без этого MAC
  не поднимается, клонируй MAC вручную в LuCI (Сеть -> Интерфейсы -> Устройства)
  до запуска playbook.
- `install.sh` в неинтерактивном режиме ставит sing-box **stable** (как на эталоне)
  и `luci-i18n-forkop-ru`.
- Forkop ставится последней версией из GitHub. Конфиг снят с 1.0.5; при выходе
  новой версии миграции выполнит сам Forkop.
- Если GitHub с роутера недоступен, установка Forkop упадёт с соответствующим
  сообщением — тогда `forkop`/`luci-app-forkop` ставятся вручную, остальное
  playbook сделает при повторном запуске.
