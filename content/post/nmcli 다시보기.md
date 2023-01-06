+++
title = "nmcli 다시보기"
tags = [
  "linux",
  "nmcli",
  "network"
]
categories = [
  "linux"
]
date = "2016-11-21T18:06:53+09:00"

+++

### 네트워크 설정
네트워크 관리를 위한 CLI (command line tool)인 nmcli 에 대해서 다시 한번 보자. (맨날 까먹으니까..... ㅠㅠ)

command line tool 인 nmcli 는 사용자가 직접 또는 NetworkManager를 위한 스크립트를 사용할 수 있다.
기본적인 명령어 모양은 아래와 같다
```bash
$ netcli [OPTIONS] OBJECT { COMMAND | help }
```
**OBJECT** 는 **general, networking, radio, connection, device, agent, monitor** 이다. 그리고 가장 많이 사용하는 **OPTIONS**은 **-t (--terse), -p (--pretty), -h (--help)** 이다.

```bash
$ nmcli help
Usage: nmcli [OPTIONS] OBJECT { COMMAND | help }

OPTIONS
  -t[erse]                                   terse output
  -p[retty]                                  pretty output
  -m[ode] tabular|multiline                  output mode
  -c[olors] auto|yes|no                      whether to use colors in output
  -f[ields] <field1,field2,...>|all|common   specify fields to output
  -e[scape] yes|no                           escape columns separators in values
  -a[sk]                                     ask for missing parameters
  -s[how-secrets]                            allow displaying passwords
  -w[ait] <seconds>                          set timeout waiting for finishing operations
  -v[ersion]                                 show program version
  -h[elp]                                    print this help

OBJECT
  g[eneral]       NetworkManager's general status and operations
  n[etworking]    overall networking control
  r[adio]         NetworkManager radio switches
  c[onnection]    NetworkManager's connections
  d[evice]        devices managed by NetworkManager
  a[gent]         NetworkManager secret agent or polkit agent
  m[onitor]       monitor NetworkManager changes
```
```bash
$ nmcli general help
COMMAND := { status | hostname | permissions | logging }

  status

  hostname [<hostname>]

  permissions

  logging [level <log level>] [domains <log domains>]
```
NetworkManager의 상태
```
$ nmcli general status
STATE      CONNECTIVITY  WIFI-HW  WIFI     WWAN-HW  WWAN    
connected  full          enabled  enabled  enabled  enabled
```
모든 connections
```
$ nmcli connection show
NAME                         UUID                                  TYPE             DEVICE
clear-guest                  3f1408fd-b085-4949-95dd-6a96f0eb2d46  802-11-wireless  wlan0  
#SFO FREE WIFI               efcce222-c7ec-491d-84dd-9b872038d555  802-11-wireless  --     
AirportWiFi                  a4c579ba-dfe8-42c8-8d5c-5883ed9468be  802-11-wireless  --     
AirportWiFi(5G)              ce323b55-87b1-49cb-a488-8b790fe34148  802-11-wireless  --     
Alt Guests                   a61ec0fc-2550-4811-89fe-a95862454999  802-11-wireless  --     
AppleNamoo_5G                9c51380f-a129-492b-8ad0-7847bf2057d2  802-11-wireless  --     
Auto Ethernet (DNS)          15ded4e9-6046-4a51-9002-a30320dda81a  802-3-ethernet   --     
Auto Ethernet (without DNS)  0550fea5-1829-4525-8eb4-ed40e80fc112  802-3-ethernet   --     
BELLALIANT1083               6005f4f4-0c46-46d9-ad70-fae14878799f  802-11-wireless  --     
BWWainwrightInnAndSuites     2c04eea6-9c16-4cc2-b9bc-df2030b51c71  802-11-wireless  --     
Coex free wifi zone          b62f6437-1ef0-4e60-9357-3e847c74b5d4  802-11-wireless  --     
DD                           9851813f-1c7c-4c9c-b79b-9c6559fc4b82  802-11-wireless  --     
DIRECT-4c-HP M277 LaserJet   a44ab5bb-1a52-4d42-8cce-6daff0d5d830  802-11-wireless  --     
Dell M900HD 9914             b3c3d2d6-b309-4090-8a1a-80a547d8e6c1  802-11-wireless  --     
Dell M900HD 991a             61db4250-7775-4dee-9f61-1a90bdfdd48e  802-11-wireless  --     
Delta-Internet               810e2710-848c-4f62-9620-3964991f64ae  802-11-wireless  --     
Desperado 5G                 1a3d4301-f35b-4a31-8587-67b9436fe7fc  802-11-wireless  --     
EDIYA_5G                     01a1efa4-0cb1-4232-bbeb-0dfe37e10819  802-11-wireless  --     
FARMTOCUP02                  59282bbf-45e6-402e-a97e-65ced51734f7  802-11-wireless  -
```
현재 접속중인 connection
```
$ nmcli connection show --active
NAME         UUID                                  TYPE             DEVICE
clear-guest  3f1408fd-b085-4949-95dd-6a96f0eb2d46  802-11-wireless  wlan0
```
디바이스 상태
```
$ nmcli device status
DEVICE   TYPE      STATE         CONNECTION  
wlan0    wifi      connected     clear-guest
eth0     ethernet  disconnected  --          
docker0  bridge    unmanaged     --          
lo       loopback  unmanaged     --  
```
Command는 몇가지 option과 object들에 대해서 줄여서 표현할 수 있다.
```
$ nmcli connection modify id 'MyCafe' 802-11-wireless.mtu 1350
-->
$ nmcli con mod MyCafe 802-11-wireless.mtu 1350

$ nmcli connection add type ethernet
-->
$ nmcli c a type eth
```

### nmcli 를 사용해서 인터페이스 시작/중지
nmcli 툴은 어떤 네트워크 인터페이스(nic)도 시작/중지 할 수 있다.

```
nmcli con up id bond0
nmcli con up id port0
nmcli dev disconnect bond0
nmcli dev disconnect ens3
```

nmcli con up my-office

### WiFi Connection
사용가능한 Wi-Fi access point 확인
```
$ nmcli dev wifi list
*  SSID                   MODE    CHAN  RATE       SIGNAL  BARS  SECURITY         
   ollehWiFi              Infra   1     54 Mbit/s  94      ▂▄▆█  WPA1 WPA2 802.1X
   ollehWiFi              Infra   1     54 Mbit/s  94      ▂▄▆█                   
   clear                  Infra   1     54 Mbit/s  84      ▂▄▆█                   
   ollehWiFi              Infra   1     54 Mbit/s  82      ▂▄▆█                   
   ollehWiFi              Infra   1     54 Mbit/s  82      ▂▄▆█  WPA1 WPA2 802.1X
   aaa_msjo_403           Infra   1     54 Mbit/s  80      ▂▄▆_  WPA1 WPA2        
   ollehWiFi              Infra   100   54 Mbit/s  77      ▂▄▆_  WPA1 WPA2 802.1X
   olleh GiGA WiFi        Infra   100   54 Mbit/s  75      ▂▄▆_                   
   ollehEgg_423           Infra   9     54 Mbit/s  74      ▂▄▆_  WPA1 WPA2        
   ollehWiFi              Infra   100   54 Mbit/s  74      ▂▄▆_                   
   SKH_ollehEgg_498       Infra   5     54 Mbit/s  69      ▂▄▆_  WPA1 WPA2
```

새로운 Wi-Fi Connection을 static-ip, dns는 자동 으로 만들기
```
$ nmcli con add con-name MyCafe ifname wlan0 type wifi ssid MyCafe ip4 192.168.100.101/24 gw4 192.168.100.1
```
WPA2 password = caffeine 를 지정
```
$ nmcli con modify MyCafe wifi-sec.key-mgme wpa-psk
$ nmcli con mofidy MyCafe wifi-sec.psk caffeine
```
