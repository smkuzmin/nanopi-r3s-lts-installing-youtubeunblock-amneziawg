## NanoPi R3S LTS: Установка youtubeUnblock


### Ссылки

- [Wiki](https://wiki.friendlyelec.com/wiki/index.php/NanoPi_R3S#Introduction)
- [Wiki - Unbricking](https://wiki.friendlyelec.com/wiki/index.php/NanoPi_R3S#Windows_Users)
- [Wiki - Flash Official OS to eMMC](https://wiki.friendlyelec.com/wiki/index.php/NanoPi_R3S?spm=a2ty_o01.29997173.0.0.46995171Rp4ie1#Flash_Official_OS_to_eMMC)
- [NanoPi-R3S - free download (Google Drive)](https://drive.google.com/drive/folders/17DfzT1JBvd3PigcOa0Rr05a0VygcL1cO)
- [4PDA](https://4pda.to/forum/index.php?showtopic=1098192&st=600)
- [Поддержка OpenWRT для NanoPi R3S LTS](https://forum.openwrt.org/t/openwrt-support-for-nanopi-r3s-lts/237172/7)

### Дефолтные настройки предустановленной на eMMC FriendlyWrt

```
LAN IP: 192.168.2.1
  USER: root
  PASS: password
```

### Сброс FriendlyWrt к дефолтным настройкам

```bash
# System -> Backup / Flash Firmware -> Restore -> Reset to defaults -> Perform reset -> OK
firstboot -y && reboot
```

### Скрипт установки youtubeUnblock на FriendlyWrt

```bash
#!/bin/bash

# Устанавливаем английский язык интерфейса
# System -> System -> Language and Style -> Language: English
uci set luci.main.lang='en' && uci commit luci && /etc/init.d/rpcd restart

# Устанавливаем часовой пояс
# System -> System -> General Settings -> Timezone: Europa/Samara -> Save & Apply
uci set system.@system[0].timezone='<+04>-4' && \
uci set system.@system[0].zonename='Europe/Samara' && \
uci commit system && /etc/init.d/system restart

# Удаляем ненужные пакеты. Оставляем в Services только:
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

# Отключаем ненужное в автозагрузке
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
/etc/init.d/wpad disable
/etc/init.d/fa-fancontrol disable
/etc/init.d/radius disable
/etc/init.d/odhcpd disable
/etc/init.d/cron disable
/etc/init.d/vsftpd disable
/etc/init.d/avahi-daemon disable
/etc/init.d/blockd disable
/etc/init.d/collectd disable
/etc/init.d/miniupnpd disable
/etc/init.d/watchcat disable
/etc/init.d/samba4 disable
/etc/init.d/wsdd2 disable

# Разрешаем подключения на WAN-интерфейсе
# Network -> Firewall -> Zones -> на пересечении wan и Input выбираем accept и жмем Save & Apply.
uci set firewall.@zone[1].input='ACCEPT'
uci commit firewall
/etc/init.d/firewall restart

# Отключаем Routing/NAT Offloading
# Flow Offloading должен быть выключен для работы любых DPI-обходчиков на базе nfqws
# Network -> Firewall -> Routing/NAT Offloading -> Flow offloading type: None
uci set firewall.@defaults[0].flow_offloading='0'
uci set firewall.@defaults[0].flow_offloading_hw='0'
uci commit firewall
/etc/init.d/firewall restart

# Устанавливаем youtubeUnblock
# System -> Software -> Upload Package -> Browse -> youtubeUnblock-*.ipk          -> Upload -> Install -> Dismiss
# System -> Software -> Upload Package -> Browse -> luci-app-youtubeUnblock-*.ipk -> Upload -> Install -> Dismiss
cd /tmp
wget -qO youtubeUnblock.ipk "https://github.com/Waujito/youtubeUnblock/releases/download/v1.3.0/youtubeUnblock-1.3.0-1-330efb5-aarch64_generic-openwrt-24.10.ipk"
wget -qO luci-app-youtubeUnblock.ipk "https://github.com/Waujito/youtubeUnblock/releases/download/v1.3.0/luci-app-youtubeUnblock-1.3.0-1-330efb5.ipk"
opkg install --force-depends ./youtubeUnblock.ipk ./luci-app-youtubeUnblock.ipk
/etc/init.d/youtubeUnblock enable
/etc/init.d/youtubeUnblock start

# Отключаем IPv6
uci set network.lan.ipv6='off'
uci set network.lan.ip6assign=''
uci set network.lan.delegate='0'
uci set network.wan.ipv6='0'
uci set network.wan.delegate='0'
uci -q get network.wan6 && uci delete network.wan6
uci commit network
ifdown wan && ifup wan

# Отключаем DHCPv6/RA в LAN
uci set dhcp.lan.dhcpv6='disabled'
uci set dhcp.lan.ndp='disabled'
uci set dhcp.lan.ra='disabled'
uci commit dhcp
/etc/init.d/odhcpd stop    2>/dev/nul
/etc/init.d/odhcpd disable 2>/dev/nul

# Перезагружаемся
reboot
```
