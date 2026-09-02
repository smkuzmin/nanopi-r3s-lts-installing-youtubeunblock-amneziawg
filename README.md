## NanoPi R3S LTS: Установка youtubeUnblock или клиента AmneziaWG

## Установка youtubeUnblock


### 1. Подключаем устройство

1. Втыкаем **LAN**-кабель (на котором по DHCP раздается Интернет), в **WAN**-порт нашего устройства.
2. Подключаем **LAN**-порт нашего устройства к ПК. Дефолтные настройки подключения **FriendlyWrt**:
```ini
WAN IP: Dynamic
LAN IP: 192.168.2.1
  USER: root
  PASS: password
```
3. Подключаемся к устройству по протоколу **SSH** через терминал [PuTTY](https://the.earth.li/~sgtatham/putty/latest/w32/putty.exe):
```powershell
   putty.exe 192.168.2.1 -l root -pw password
```
4. Следующие этапы подразумевают подключение к устройству через терминал и выполнение в нем указанных блоков кода.

### 2. Сбрасываем OpenWrt к дефолтным настройкам и перезагружаемся

```bash
### Сбрасываем OpenWrt к дефолтным настройкам и перезагружаемся
# System -> Backup / Flash Firmware -> Restore -> Reset to defaults -> Perform reset -> OK
firstboot -y && reboot
```

### 3. Подготавливаем дефолтную OpenWrt и перезагружаемся

```bash
### Устанавливаем английский язык интерфейса
# System -> System -> Language and Style -> Language: English
uci set luci.main.lang='en'

### Устанавливаем часовой пояс
# System -> System -> General Settings -> Timezone: Europe/Samara
uci set system.@system[0].timezone='<+04>-4'
uci set system.@system[0].zonename='Europe/Samara'

### Отключаем PoE-Out на MikroTik (чтобы порт не горел красным) - добавляем команды (перед exit 0) в скрипт автозапуска
# System -> Startup -> Local Startup:
# sleep 2; for f in /sys/class/gpio/*poe*/value; do echo 0 >$f; done
# exit 0
# -> Save -> Dismiss
grep -q 'gpio.*poe' /etc/rc.local || sed -i '/exit 0/i sleep 2; for f in /sys/class/gpio/*poe*/value; do echo 0 >$f; done' /etc/rc.local

### Разрешаем подключения на WAN-интерфейсе
# Network -> Firewall -> Zones -> at the intersection of wan and Input, select accept -> Save & Apply
uci set firewall.@zone[1].input='ACCEPT'

### Отключаем IPv6
# Удаляем IPv6-туннели и интерфейсы
# Network -> Interfaces -> удаляем WAN6, 6in4, 6to4
uci -q delete network.wan6
uci -q delete network.6in4
uci -q delete network.6to4
# Отключаем IPv6 на LAN-интерфейсе
# Network -> Interfaces -> LAN -> Advanced Settings
uci    set    network.lan.ipv6='off'
uci    set    network.lan.delegate='0'
uci -q delete network.lan.ip6assign
uci -q delete network.lan.ip6addr
uci -q delete network.lan.ip6prefix
# Отключаем IPv6 на WAN-интерфейсе
# Network -> Interfaces -> WAN -> Advanced Settings
uci    set    network.wan.ipv6='0'
uci    set    network.wan.delegate='0'
uci -q delete network.wan.ip6addr
uci -q delete network.wan.ip6prefix
# Отключаем IPv6 в DHCP
uci    set    dhcp.lan.dhcpv6='disabled'
uci    set    dhcp.lan.ndp='disabled'
uci    set    dhcp.lan.ra='disabled'
uci -q delete dhcp.wan.dhcpv6
uci -q delete dhcp.wan.ra
# Удаляем IPv6-правила в Firewall
for rule in $(uci show firewall|grep "family='ipv6'"|cut -d'.' -f2|cut -d'=' -f1|sort -t'[' -k2 -rn); do uci delete firewall.$rule; done
for rule in $(uci show firewall|grep 'ip6proto='|cut -d'.' -f2|cut -d'=' -f1); do uci delete firewall.$rule.ip6proto; done
for rule in $(uci show firewall|grep '@rule'|grep -v 'family='|cut -d'[' -f2|cut -d']' -f1); do uci set firewall.@rule[$rule].family='ipv4'; done
# Отключаем IPv6 в sysctl
grep -q 'net.ipv6.conf.all.disable_ipv6=1'     /etc/sysctl.conf || echo >>/etc/sysctl.conf 'net.ipv6.conf.all.disable_ipv6=1'
grep -q 'net.ipv6.conf.default.disable_ipv6=1' /etc/sysctl.conf || echo >>/etc/sysctl.conf 'net.ipv6.conf.default.disable_ipv6=1'
grep -q 'net.ipv6.conf.lo.disable_ipv6=1'      /etc/sysctl.conf || echo >>/etc/sysctl.conf 'net.ipv6.conf.lo.disable_ipv6=1'
grep -q 'net.ipv6.conf.br-lan.disable_ipv6=1'  /etc/sysctl.conf || echo >>/etc/sysctl.conf 'net.ipv6.conf.br-lan.disable_ipv6=1'
# Удаляем IPv6-пакеты (удаляем пакеты, и все зависящие от них)
opkg remove --force-removal-of-dependent-packages odhcp6c
opkg remove --force-removal-of-dependent-packages odhcpd-ipv6only
opkg remove --force-removal-of-dependent-packages kmod-ip6tables
opkg remove --force-removal-of-dependent-packages kmod-nf-nat6
opkg remove --force-removal-of-dependent-packages kmod-ipt-nat6
opkg remove --force-removal-of-dependent-packages ip6tables
opkg remove --force-removal-of-dependent-packages luci-proto-ipv6

### Удаляем ненужные пакеты (не обращая внимания на зависимости)
# Оставляем в Services только:
#  - Kernel Manager
#  - QoS over Nftables
#  - youtubeUnblock
#  - Bandwith Monitor
#  - Watchcat
#  - Network Shares
#  - Terminal
#  - UPnP IGD & PCP
opkg remove --force-depends adblock  luci-app-adblock
opkg remove --force-depends aria2    luci-app-aria2
opkg remove --force-depends ddns     luci-app-ddns
opkg remove --force-depends hd-idle  luci-app-hd-idle
opkg remove --force-depends minidlna luci-app-minidlna
opkg remove --force-depends netdata  luci-app-netdata
opkg remove --force-depends qos      luci-app-qos
opkg remove --force-depends smartdns luci-app-smartdns

### Отключаем ненужные сервисы в автозагрузке
# System -> Startup:
#
# Pr  Имя              Описание                                   Можно ли отключать
# --  ---------------  -----------------------------------------  -------------------------------------------------------------------------
# 19  wpad             WPA-Enterprise / 802.1X                    Да, если используется только WPA2-Personal
# 21  fa-fancontrol    Управление вентилятором                    Да, если роутер с пассивным охлаждением
# 30  radius           RADIUS-клиент                              Да, если не корпоративная сеть 802.1X
# 35  odhcpd           DHCPv6 и RA для IPv6                       Да, если IPv6 отключён (рекомендуется для youtubeUnblock)
# 50  cron             Планировщик задач                          Да, если не используете расписания
# 50  vsftpd           FTP-сервер                                 Да, если не используется FTP
# 61  avahi-daemon     mDNS/Bonjour (локальное обнаружение)       Да, если не используете AirPlay/Chromecast discovery
# 80  blockd           Авто-монтирование USB-накопителей          Да, если нет USB-дисков
# 80  collectd         Сбор метрик для мониторинга                Да, если не используете внешние системы мониторинга
# 94  miniupnpd        Автоматический проброс портов (UPnP)       Да, если не используете торренты или устройства, которые сами открывают порты (IP-камеры, NAS и т.п.)
# 97  watchcat         Автоматическая перезагрузка при зависании  Да, если не используется Watchcat
# 98  samba4           SMB-сервер для общего доступа к файлам     Да, если не расшариваете папки по сети
# 99  wsdd2            Web Service Discovery для SMB              Да, если не нужен в Windows-сети
# --  ---------------  -----------------------------------------  -------------------------------------------------------------------------
# 50  sqm              Smart Queue Management (QoS)               Да, если не используете торренты/видеозвонки на перегруженном канале
# 60  nlbwmon          Мониторинг трафика по хостам               Да, если не нужен Bandwith Monitor
# 79  luci_statistics  Статистика в веб-интерфейсе                Да, если не смотрите графики в LuCI
/etc/init.d/avahi-daemon   stop 2>/dev/null && /etc/init.d/avahi-daemon   disable
/etc/init.d/blockd         stop 2>/dev/null && /etc/init.d/blockd         disable
/etc/init.d/collectd       stop 2>/dev/null && /etc/init.d/collectd       disable
/etc/init.d/cron           stop 2>/dev/null && /etc/init.d/cron           disable
/etc/init.d/fa-fancontrol  stop 2>/dev/null && /etc/init.d/fa-fancontrol  disable
/etc/init.d/miniupnpd      stop 2>/dev/null && /etc/init.d/miniupnpd      disable
/etc/init.d/odhcpd         stop 2>/dev/null && /etc/init.d/odhcpd         disable
/etc/init.d/radius         stop 2>/dev/null && /etc/init.d/radius         disable
/etc/init.d/samba4         stop 2>/dev/null && /etc/init.d/samba4         disable
/etc/init.d/vsftpd         stop 2>/dev/null && /etc/init.d/vsftpd         disable
/etc/init.d/watchcat       stop 2>/dev/null && /etc/init.d/watchcat       disable
/etc/init.d/wpad           stop 2>/dev/null && /etc/init.d/wpad           disable
/etc/init.d/wsdd2          stop 2>/dev/null && /etc/init.d/wsdd2          disable
/etc/init.d/youtubeUnblock stop 2>/dev/null && /etc/init.d/youtubeUnblock disable

### Применяем все изменения
uci commit
sysctl -p -q

### Перезагружаемся
# System -> Reboot -> Perform reboot
reboot
```

### 4. Устанавливаем youtubeUnblock

```bash
(
  ### Обновляем списки пакетов
  # System -> Software -> Update lists..
  opkg update || { echo 'OPKG UPDATE ERROR'; exit 1; }

  ### Устанавливаем зависимости для youtubeUnblock
  # System -> Software -> Download and install package: kmod-nfnetlink-queue -> OK -> Install -> Dismiss
  # System -> Software -> Download and install package: kmod-nft-queue       -> OK -> Install -> Dismiss
  # System -> Software -> Download and install package: kmod-nf-conntrack    -> OK -> Install -> Dismiss
  opkg install kmod-nfnetlink-queue kmod-nft-queue kmod-nf-conntrack

  ### Скачиваем и устанавливаем пакеты youtubeUnblock для OpenWrt 24.10
  # System -> Software -> Upload Package.. -> Browse.. -> youtubeUnblock-1.3.1-1-4a223b0-aarch64_generic-openwrt-24.10.ipk -> Upload -> Install -> Dismiss
  # System -> Software -> Upload Package.. -> Browse.. -> luci-app-youtubeUnblock-1.3.1-1-4a223b0.ipk -> Upload -> Install -> Dismiss
   VERSION='1.3.1'
  BASE_URL="https://github.com/Waujito/youtubeUnblock/releases/download/v${VERSION}"
     BUILD="1-4a223b0"
      ARCH=$(opkg print-architecture|awk 'END{print $2}')
  pkg="youtubeUnblock-${VERSION}-${BUILD}-${ARCH}-openwrt-24.10"
  url="${BASE_URL}/${pkg}.ipk"
  wget -qO     "/tmp/${pkg}.ipk" "${url}" || { echo "DOWNLOAD ERROR: ${url}"; exit 1; }
  opkg install "/tmp/${pkg}.ipk"          || { echo "INSTALL ERROR: ${pkg}" ; exit 1; }
  rm -f        "/tmp/${pkg}.ipk"
  pkg="luci-app-youtubeUnblock-${VERSION}-${BUILD}"
  url="${BASE_URL}/${pkg}.ipk"
  wget -qO     "/tmp/${pkg}.ipk" "${url}" || { echo "DOWNLOAD ERROR: ${url}"; exit 1; }
  opkg install "/tmp/${pkg}.ipk"          || { echo "INSTALL ERROR: ${pkg}" ; exit 1; }
  rm -f        "/tmp/${pkg}.ipk"

  ### Отключаем Routing/NAT Offloading (он должен быть выключен для работы любых DPI-обходчиков на базе nfqws)
  # Network -> Firewall -> Routing/NAT Offloading -> Flow offloading type: None
  uci set firewall.@defaults[0].flow_offloading='0'
  uci set firewall.@defaults[0].flow_offloading_hw='0'

  ### Применяем изменения
  uci commit
  /etc/init.d/firewall restart

  ### Включаем youtubeUnblock в автозагрузку
  # System -> Startup -> youtubeUnblock -> Enabled
  /etc/init.d/youtubeUnblock enable

  ### Запускаем youtubeUnblock
  /etc/init.d/youtubeUnblock restart
)
```

Далее выходим из веб-интерфейса (**Log out**) и входим заново. В меню **Services** появится новый пункт **youtubeUnblock**.

Если ютуб еще не заработал, то мне (провайдер Ростелеком) помогло это: **Services** -> **youtubeUnblock** -> **Configuration** -> **Default section** -> **Edit** -> **\[ \] Fake sni** -> **Save** -> **Save & Apply**.

Вот и все - теперь YouTube работает без использования VPN. И еще бонусом - в YouTube не будет рекламы.

***

## Установка клиента AmneziaWG


### 1. Устанавливаем клиент AmneziaWG и перезагружаемся

```bash
(
  ### Обновляем списки пакетов
  # System -> Software -> Update lists..
  opkg update || { echo 'ERROR: opkg update'; exit 1; }

  ### Устанавливаем необходимые зависимости для AmneziaWG
  # System -> Software -> Download and install package: kmod-udptunnel4                  -> OK -> Install -> Dismiss
  # System -> Software -> Download and install package: kmod-udptunnel6                  -> OK -> Install -> Dismiss
  # System -> Software -> Download and install package: kmod-crypto-lib-chacha20poly1305 -> OK -> Install -> Dismiss
  # System -> Software -> Download and install package: kmod-crypto-lib-curve25519       -> OK -> Install -> Dismiss
  # System -> Software -> Download and install package: kmod-crypto-hash                 -> OK -> Install -> Dismiss
  # System -> Software -> Download and install package: kmod-crypto-aead                 -> OK -> Install -> Dismiss
  opkg install kmod-udptunnel4 kmod-udptunnel6 kmod-crypto-lib-chacha20poly1305 kmod-crypto-lib-curve25519 kmod-crypto-hash kmod-crypto-aead \
  || { echo 'ERROR: opkg install AmneziaWG depends'; exit 1; }

  ### Скачиваем и устанавливаем модуль ядра, утилиты и LuCI-интерфейс AmneziaWG
  # System -> Software -> Upload Package.. -> Browse.. -> kmod-amneziawg_*.ipk       -> Upload -> Install -> Dismiss
  # System -> Software -> Upload Package.. -> Browse.. -> amneziawg-tools_*.ipk      -> Upload -> Install -> Dismiss
  # System -> Software -> Upload Package.. -> Browse.. -> luci-proto-amneziawg_*.ipk -> Upload -> Install -> Dismiss
  download_and_install()
  {
    wget -qO "/tmp/$1.ipk" "$2" || { echo "ERROR: Download url: $2"; exit 1; }
    opkg install --force-downgrade --force-depends "/tmp/$1.ipk" || { echo "ERROR: Install pkg: $1" ; exit 1; }
    rm -f "/tmp/$1.ipk"
  }
  ARCH=$(opkg print-architecture|awk 'END{print $2}')
  KERNEL=$(uname -r|cut -d. -f1,2)
  case "$ARCH" in
    mips_24kc)  # MikroTik на платформе MIPSBE
      V="24.10.8"; B="https://github.com/Slava-Shchipunov/awg-openwrt/releases/download/v${V}"; A="mips_24kc_ath79_mikrotik"
      download_and_install "kmod-amneziawg"        "${B}/kmod-amneziawg_v${V}_${A}.ipk"
      download_and_install "amneziawg-tools"       "${B}/amneziawg-tools_v${V}_${A}.ipk"
      download_and_install "luci-proto-amneziawg"  "${B}/luci-proto-amneziawg_v${V}_${A}.ipk" ;;
    aarch64_cortex-a53)  # NanoPi R3S LTS
      case "$KERNEL" in
        "6.1") KV="1.0.20260611"; KB="https://github.com/lastharbor/kmod-amneziawg-nanopi-r5c/releases/download/v${KV}-r1" ;;
        "6.6") KV="3.1.20260812"; KB="https://github.com/lastharbor/kmod-amneziawg-nanopi-r5c/releases/download/v${KV}"    ;;
            *) echo "ERROR: Unsupported kernel version: $KERNEL"; exit 1 ;;
      esac
      KV="$KV-r1"                                                                                ; KA="aarch64_generic"
      UV="24.10.8"; UB="https://github.com/Slava-Shchipunov/awg-openwrt/releases/download/v${UV}"; UA="aarch64_generic_rockchip_armv8"
      download_and_install "kmod-amneziawg"       "${KB}/kmod-amneziawg_${KV}_${KA}.ipk"
      download_and_install "amneziawg-tools"      "${UB}/amneziawg-tools_v${UV}_${UA}.ipk"
      download_and_install "luci-proto-amneziawg" "${UB}/luci-proto-amneziawg_v${UV}_${UA}.ipk" ;;
    *) echo "ERROR: Unsupported architecture: $ARCH"; exit 1 ;;
  esac

  ### Перезагружаемся
  # System -> Reboot -> Perform reboot
  echo 'Installation succesfull. Rebooting...'
  reboot
)
```

### 2. Настраиваем клиент AmneziaWG (через веб-интерфейс)

 - **Network** -> **Interfaces** -> **Add new interface..** -> Name: `awg0`, Protocol: `AmneziaWG VPN` -> **Create interface**
 - Import configuration: **Load configuration..**
 - Скачиваем конфиг AmneziaWG с сайта [WARP Генератор](https://warp-generation.github.io/) и вставляем содержимое в пустое поле
 - **Import settings** -> **OK**
 - **Firewall Settings** -> Create / Assign firewall-zone: `wan`
 - **Peers** -> **Edit** -> Persistent Keepalive: `25`
 - **Save** -> **Save** -> **Save & Apply**

### 3. Донастраиваем клиент AmneziaWG и маршрутизацию для него

```bash
(
  ### Добавляем интерфейс awg0 в зону WAN Firewall
  # Network -> Interfaces -> awg0 -> Edit -> Create / Assign firewall-zone: wan -> Save -> Save & Apply
  uci -q get firewall.@zone[1].network|grep -q 'awg0' || uci add_list firewall.@zone[1].network='awg0'

  ### Устанавливаем Persistent Keep Alive для поддержания NAT-сессии у провайдера, иначе handshake может не быть
  # Network -> Interfaces -> awg0 -> Edit -> Peers -> Edit -> Persistent Keep Alive: 25 -> Save -> Save -> Save & Apply
  uci set network.awg0.persistent_keepalive='25'
  uci -q get network.@amneziawg_awg0[0] >/dev/null && uci set network.@amneziawg_awg0[0].persistent_keepalive='25'

  ### Применяем изменения
  uci commit
  ifdown awg0; sleep 3; ifup awg0

  ### Проверяем, что AWG-туннель работает (ждем до 30 секунд)
  awg_is_alive() { awg show awg0 2>/dev/null|grep handshake|grep -qvi never; }
  for i in $(seq 1 30); do sleep 1; awg_is_alive && break; done
  awg_is_alive || { echo 'ERROR: awg0 - no handshake! Check VPN connection'; exit 1; }

  ### Настраиваем маршрутизацию
  # Трафик от клиентов (in='lan') в приватные сети отправляем в таблицу main (LAN/WAN-интерфейсы). За это отвечают:
  #   - правило с приоритетом 10: in='lan' -> 10.0.0.0/8     => таблица main
  #   - правило с приоритетом 10: in='lan' -> 172.16.0.0/12  => таблица main
  #   - правило с приоритетом 10: in='lan' -> 192.168.0.0/16 => таблица main
  #   - маршруты, динамически создаваемые LAN/WAN-интерфейсами в таблице main
  # Остальной трафик от клиентов (in='lan') - в Интернет, его отправляем в таблицу 100 (AWG-туннель). За это отвечают:
  #   - правило c приоритетом 20: in='lan' -> ANY => таблица 100
  #   - маршрут по умолчанию через интерфейс awg0 в таблице 100
  # Трафик от самого роутера (in='lo') идет через LAN/WAN-интерфейсы (нужно для обновления времени, DNS, и т.д.). За это отвечают:
  #   - маршруты, динамически создаваемые LAN/WAN-интерфейсами в таблице main
  # Примечания:
  #   in='' - входящий интерфейс       (UCI), iif='' - то же в ядре Linux (его показывает ip rule show)
  #   lan   - логическое имя LAN-моста (UCI), br-lan - то же в ядре Linux (его показывает ip rule show)

  ### Добавляем маршрут по умолчанию через интерфейс awg0 в таблице 100
  # Network -> Routing -> Add -> Interface: awg0, Route type: unicast, Target: 0.0.0.0/0 -> Advanced Settings -> Table: 100 -> Save -> Save & Apply
  # Перед добавлением маршрута: Удаление всех старых маршрутов через интерфейс awg0
  uci show network|grep 'route.*awg0'|cut -d. -f2|sort -Vr|while read r; do uci -q delete network.$r; done
  uci add network route >/dev/null
  uci set network.@route[-1].interface='awg0'
  uci set network.@route[-1].target='0.0.0.0/0'
  uci set network.@route[-1].table='100'

  ### Добавляем правила с приоритетом 10: Трафик от клиентов (in='lan') в приватные сети отправляем в таблицу main (LAN/WAN-интерфейсы)
  # Network -> Routing -> IPv4 Rules -> Add -> Priority: 10, Incoming interface: lan, Destination: 10.0.0.0/8     -> Save -> Save & Apply
  # Network -> Routing -> IPv4 Rules -> Add -> Priority: 10, Incoming interface: lan, Destination: 172.16.0.0/12  -> Save -> Save & Apply
  # Network -> Routing -> IPv4 Rules -> Add -> Priority: 10, Incoming interface: lan, Destination: 192.168.0.0/16 -> Save -> Save & Apply
  # Перед добавлением правил: Удаление всех старых правил для входящего интерфейса lan
  uci show network|grep "rule.*\.in='lan'"|cut -d. -f2|sort -Vr|while read r; do uci -q delete network.$r; done
  uci add network rule >/dev/null
  uci set network.@rule[-1].in='lan'
  uci set network.@rule[-1].dest='10.0.0.0/8'
  uci set network.@rule[-1].lookup='main'
  uci set network.@rule[-1].priority='10'
  uci add network rule >/dev/null
  uci set network.@rule[-1].in='lan'
  uci set network.@rule[-1].dest='172.16.0.0/12'
  uci set network.@rule[-1].lookup='main'
  uci set network.@rule[-1].priority='10'
  uci add network rule >/dev/null
  uci set network.@rule[-1].in='lan'
  uci set network.@rule[-1].dest='192.168.0.0/16'
  uci set network.@rule[-1].lookup='main'
  uci set network.@rule[-1].priority='10'

  ### Добавляем правило с приоритетом 20: Остальной трафик (в Интернет) от клиентов (in='lan') отправляем в таблицу 100 (AWG-туннель)
  # Network -> Routing -> IPv4 Rules -> Add -> Priority: 20, Incoming interface: lan -> Advanced Settings -> Table: 100 -> Save -> Save & Apply
  uci add network rule >/dev/null
  uci set network.@rule[-1].in='lan'
  uci set network.@rule[-1].lookup='100'
  uci set network.@rule[-1].priority='20'

  ### Применяем изменения
  uci commit
  /etc/init.d/network reload
  /etc/init.d/firewall reload
  sleep 5
)
```

### 4. Проверяем ключевые настройки

```bash
(
  check() { r="31m[-]"; eval "$2" &>/dev/null && r="32m[+]"; printf "\033[1;%s\033[0m %s\n" "$r" "$1"; }
  wan() { uci get network.wan.device || uci get network.wan.ifname || echo none; }
  check "  INTERNET: Интернет доступен (ping 8.8.8.8)"                              "ping -c 1 -W 5 8.8.8.8"
  check "   ROUTING: Пересылка между интерфейсами включена"                         "sysctl -n net.ipv4.ip_forward|grep 1"
  check " VPN / AWG: Интерфейс AWG добавлен в зону WAN"                             "uci get firewall.@zone[1].network|grep awg0"
  check " VPN / AWG: Параметр 'Persistent Keep Alive' включен"                      "uci get network.awg0.persistent_keepalive|grep '[1-9]'"
  check " VPN / AWG: Конфигурация импортирована (есть peer)"                        "uci show network|grep amneziawg_awg0"
  check " VPN / AWG: Соединение установлено (есть handshake)"                       "awg show awg0|grep handshake|grep -vi never"
  check " DEF ROUTE: Есть маршрут по умолчанию через WAN, и он только в main"       "ip route show table all|grep 'default .* dev `wan`\>'|grep -v table"
  check " DEF ROUTE: Есть маршрут по умолчанию через AWG, и он только в 100"        "ip route show table all|grep 'default dev awg0 table 100\>'"
  check "ROUTE RULE: Есть правило: трафик от клиентов в 10.0.0.0/8     => main"     "ip rule show|grep br-lan|grep '10.0.0.0/8.*main'"
  check "ROUTE RULE: Есть правило: трафик от клиентов в 172.16.0.0/12  => main"     "ip rule show|grep br-lan|grep '172.16.0.0/12.*main'"
  check "ROUTE RULE: Есть правило: трафик от клиентов в 192.168.0.0/16 => main"     "ip rule show|grep br-lan|grep '192.168.0.0/16.*main'"
  check "ROUTE RULE: Есть правило: трафик от клиентов в Интернет       => 100"      "ip rule show|grep br-lan|grep 'lookup 100\>'"
  check " ROUTE GET: Проверка маршрута: трафик от клиентов в 10.0.0.0/8     => LAN" "ip route get 10.0.0.1    from 192.168.1.50 iif br-lan|grep br-lan"
  check " ROUTE GET: Проверка маршрута: трафик от клиентов в 172.16.0.0/12  => LAN" "ip route get 172.16.0.1  from 192.168.1.50 iif br-lan|grep br-lan"
  check " ROUTE GET: Проверка маршрута: трафик от клиентов в 192.168.0.0/16 => LAN" "ip route get 192.168.0.1 from 192.168.1.50 iif br-lan|grep br-lan"
  check " ROUTE GET: Проверка маршрута: трафик от клиентов в Интернет       => AWG" "ip route get 8.8.8.8     from 192.168.1.50 iif br-lan|grep awg0"
  check "  NTP SYNC: Время синхронизировано с pool.ntp.org"                         "ntpd -n -q -p pool.ntp.org"
)
```

***

### Ссылки

- [Wiki](https://wiki.friendlyelec.com/wiki/index.php/NanoPi_R3S#Introduction)
- [Wiki - Unbricking](https://wiki.friendlyelec.com/wiki/index.php/NanoPi_R3S#Windows_Users)
- [Wiki - Flash Official OS to eMMC](https://wiki.friendlyelec.com/wiki/index.php/NanoPi_R3S?spm=a2ty_o01.29997173.0.0.46995171Rp4ie1#Flash_Official_OS_to_eMMC)
- [NanoPi-R3S - free download (Google Drive)](https://drive.google.com/drive/folders/17DfzT1JBvd3PigcOa0Rr05a0VygcL1cO)
- [4PDA](https://4pda.to/forum/index.php?showtopic=1098192&st=600)
- [Поддержка OpenWRT для NanoPi R3S LTS](https://forum.openwrt.org/t/openwrt-support-for-nanopi-r3s-lts/237172/7)
