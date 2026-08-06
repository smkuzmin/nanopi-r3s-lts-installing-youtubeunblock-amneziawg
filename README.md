## NanoPi R3S LTS: Установка youtubeUnblock

### Ссылки

- [Wiki](https://wiki.friendlyelec.com/wiki/index.php/NanoPi_R3S#Introduction)
- [Wiki - Unbricking](https://wiki.friendlyelec.com/wiki/index.php/NanoPi_R3S#Windows_Users)
- [Wiki - Flash Official OS to eMMC](https://wiki.friendlyelec.com/wiki/index.php/NanoPi_R3S?spm=a2ty_o01.29997173.0.0.46995171Rp4ie1#Flash_Official_OS_to_eMMC)
- [NanoPi-R3S - free download (Google Drive)](https://drive.google.com/drive/folders/17DfzT1JBvd3PigcOa0Rr05a0VygcL1cO)
- [4PDA](https://4pda.to/forum/index.php?showtopic=1098192&st=600)
- [Поддержка OpenWRT для NanoPi R3S LTS](https://forum.openwrt.org/t/openwrt-support-for-nanopi-r3s-lts/237172/7)

### Дефолтные настройки подключения FriendlyWrt

```
LAN IP: 192.168.2.1
  USER: root
  PASS: password
```

### 1. Сбрасываем FriendlyWrt к дефолтным настройкам

```bash
# System -> Backup / Flash Firmware -> Restore -> Reset to defaults -> Perform reset -> OK
firstboot -y && reboot
```

### 2. Подготавливаем дефолтную FriendlyWrt/OpenWrt

```bash
### Устанавливаем английский язык интерфейса
# System -> System -> Language and Style -> Language: English
uci set luci.main.lang='en'

### Устанавливаем часовой пояс
# System -> System -> General Settings -> Timezone: Europa/Samara
uci set system.@system[0].timezone='<+04>-4'
uci set system.@system[0].zonename='Europe/Samara'

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
for rule in $(uci show firewall|grep "ip6proto="|cut -d'.' -f2|cut -d'=' -f1); do uci delete firewall.$rule.ip6proto; done
for rule in $(uci show firewall|grep "@rule"|grep -v "family="|cut -d'[' -f2|cut -d']' -f1); do uci set firewall.@rule[$rule].family='ipv4'; done
# Отключаем IPv6 в sysctl
grep -q "net.ipv6.conf.all.disable_ipv6=1"     /etc/sysctl.conf || echo >>/etc/sysctl.conf "net.ipv6.conf.all.disable_ipv6=1"
grep -q "net.ipv6.conf.default.disable_ipv6=1" /etc/sysctl.conf || echo >>/etc/sysctl.conf "net.ipv6.conf.default.disable_ipv6=1"
grep -q "net.ipv6.conf.lo.disable_ipv6=1"      /etc/sysctl.conf || echo >>/etc/sysctl.conf "net.ipv6.conf.lo.disable_ipv6=1"
grep -q "net.ipv6.conf.br-lan.disable_ipv6=1"  /etc/sysctl.conf || echo >>/etc/sysctl.conf "net.ipv6.conf.br-lan.disable_ipv6=1"
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
opkg remove --force-depends netdata
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
# 50  vsftpd           FTP-сервер                                 Да, если FTP не используется
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
/etc/init.d/avahi-daemon   2>/dev/null disable
/etc/init.d/blockd         2>/dev/null disable
/etc/init.d/collectd       2>/dev/null disable
/etc/init.d/cron           2>/dev/null disable
/etc/init.d/fa-fancontrol  2>/dev/null disable
/etc/init.d/miniupnpd      2>/dev/null disable
/etc/init.d/odhcpd         2>/dev/null disable
/etc/init.d/radius         2>/dev/null disable
/etc/init.d/samba4         2>/dev/null disable
/etc/init.d/vsftpd         2>/dev/null disable
/etc/init.d/watchcat       2>/dev/null disable
/etc/init.d/wpad           2>/dev/null disable
/etc/init.d/wsdd2          2>/dev/null disable
/etc/init.d/youtubeUnblock 2>/dev/null disable

### Применяем все изменения
uci commit
sysctl -p -q

### Перезагружаемся
# System -> Reboot -> Perform reboot
reboot
```

### 3. Устанавливаем youtubeUnblock

```bash
# Устанавливаем youtubeUnblock
# System -> Software -> Upload Package -> Browse -> youtubeUnblock-*.ipk          -> Upload -> Install -> Dismiss
# System -> Software -> Upload Package -> Browse -> luci-app-youtubeUnblock-*.ipk -> Upload -> Install -> Dismiss
URL="https://github.com/Waujito/youtubeUnblock/releases/download/v1.3.0"
wget -qO /tmp/youtubeUnblock.ipk          "${URL}/youtubeUnblock-1.3.0-1-330efb5-aarch64_generic-openwrt-24.10.ipk"
wget -qO /tmp/luci-app-youtubeUnblock.ipk "${URL}/luci-app-youtubeUnblock-1.3.0-1-330efb5.ipk"
ARCH=$(opkg print-architecture | awk 'END{print $2}')
echo "System architecture: $ARCH"
if ! opkg install /tmp/youtubeUnblock.ipk /tmp/luci-app-youtubeUnblock.ipk; then
  echo "ERROR: Failed to install youtubeUnblock"
  exit 1
fi
rm -f /tmp/*.ipk

### Отключаем Routing/NAT Offloading (он должен быть выключен для работы любых DPI-обходчиков на базе nfqws)
# Network -> Firewall -> Routing/NAT Offloading -> Flow offloading type: None
uci set firewall.@defaults[0].flow_offloading='0'
uci set firewall.@defaults[0].flow_offloading_hw='0'

### Разрешаем подключения на WAN-интерфейсе
# Network -> Firewall -> Zones -> at the intersection of wan and Input, select accept -> Save & Apply
uci set firewall.@zone[1].input='ACCEPT'

### Удаляем параметры Fullcone NAT из зоны WAN, чтобы убрать предупреждения firewall
uci -q delete firewall.@zone[1].fullcone4
uci -q delete firewall.@zone[1].fullcone6

### Применяем все изменения
uci commit firewall
/etc/init.d/firewall restart

# Включаем youtubeUnblock в автозагрузку
# System -> Startup -> youtubeUnblock -> Enabled
/etc/init.d/youtubeUnblock enable

# Запускаем youtubeUnblock
/etc/init.d/youtubeUnblock restart
```
