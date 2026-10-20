# Эталонный роутер 192.168.3.1

Снято `playbooks/dump.yml` / вручную 2026-10-20. Секретов здесь нет — только
характеристики, по которым собран этот playbook.

## Железо и прошивка

| Параметр | Значение |
|---|---|
| Модель | Xiaomi Mi Router AX3000T (`xiaomi,mi-router-ax3000t`) |
| Платформа | `mediatek/filogic`, aarch64_cortex-a53 |
| Прошивка | OpenWrt 24.10.7 (r29197-ab4c7d6af7) |
| Порты | `wan`, `lan2`, `lan3`, `lan4` (все LAN в `br-lan`) |
| Wi-Fi | MT7981, phy0 2.4 ГГц, phy1 5 ГГц |

## Сеть

- LAN: `br-lan` `192.168.3.1/24`, DSA-порты lan2–lan4, `ip6assign 60`, `delegate 0`
- WAN: PPPoE, IPv6 выключен, `sourcefilter 0`, MAC клонирован (`cc:d8:43:19:8f:67`)
- DHCP: `100–249`, аренда 12 ч (дефолт)
- Firewall: дефолт, DNAT/проброса портов нет

## Wi-Fi

- 2.4 ГГц: канал 1, HE20, `cell_density 0`, WPA2-PSK
- 5 ГГц: канал 36, HE80, `cell_density 0`, WPA2-PSK
- Страна не задана

## Система

- hostname `kosmos`, TZ `UTC`, NTP — пул OpenWrt
- syslog по UDP, 128 КБ, dropbear с паролем только на LAN

## DNS

- dnsmasq: `noresolv 1`, `cachesize 0`, `ednspacket_max 1232`, единственный
  upstream — `127.0.0.42` (DNS Forkop)
- эти опции выставляет сам Forkop (`/usr/lib/forkop/dns/apply.uc`), исходные
  значения он сохраняет в `forkop_noresolv` / `forkop_cachesize`

## Forkop 1.0.5

- `settings`: DNS over HTTPS, bootstrap `77.88.8.8`, DNS `1.1.1.1`,
  `prefer_ipv4`, `disable_quic 1`, список/компоненты качаются через прокси,
  `source_network_interfaces br-lan`, мониторинг WAN, `yacd` включён
- секция `main`: `connection`, сортировка по задержке, community-списки
  `russia_inside`, `youtube`, `telegram`, `github`, `cloudflare`,
  исключены страны GB/RU
- секция `russian`: RU-домены через RU-узлы, `dashboard include RU`
- подписки: 3 в `main`, 1 в `russian`, обновление 1 ч
- приоритетная группа `pg_77450153` (`main-priority`): Основные → Резерв → Напрямую,
  health-check `gstatic.com/generate_204` каждые 5 с
- URLTest: интервал 1 мин, tolerance 50, исключены GB/RU
- sing-box 1.12.22 (stable), конфиг `/etc/sing-box/config.json`

## SQM

- включён на `pppoe-wan`: `cake` + `piece_of_cake.qos`,
  download 85000 kbit, upload 10000 kbit, `linklayer none`

## Прочие пакеты

`curl`, `bind-dig`, `coreutils`, `luci-ssl`, `luci-app-sqm`, `iptables-*`
(нужны Forkop/Zapret). Playbook ставит только то, что нужно Forkop и SQM.
