# iBGP

Актуальные схемы сети и таблиц ip-адресации представлены [тут](https://github.com/DemonOfLaziness/otus-labs/tree/main/labs/lab16/Schemes).  
Файл лабораторной и основные конфиги представлены [тут](https://github.com/DemonOfLaziness/otus-labs/tree/main/labs/lab16/Configs).  

Фрагмент общей схемы, на котором будут проводиться работы:  
![](Scheme.png)  

Ход работы

- [Подготовка](#подготовка)
- [Настройка iBGP в офисе Москва между маршрутизаторами R14 и R15](#настройка-ibgp-в-офисе-москва-между-маршрутизаторами-r14-и-r15)
- [Настройка iBGP в провайдере Триада с использованием RR](#настройка-ibgp-в-провайдере-триада-с-использованием-rr)
- [Настройка офиса Москвы так, чтобы приоритетным провайдером стал Ламас](#настройка-офиса-москвы-так-чтобы-приоритетным-провайдером-стал-ламас)
- [Настройка офиса С.-Петербург так, чтобы трафик до любого офиса распределялся по двум линкам одновременно](#настройка-офиса-с-петербург-так-чтобы-трафик-до-любого-офиса-распределялся-по-двум-линкам-одновременно)
- [Настройка IP-связности всех сетей в лабораторной работе]()

## Подготовка

Перед настройкой необходимо соединить eBGP-соседством Киторн и Триаду (R22 <> R23).  

Настройка на R22:  
```
R22(config-router)#neighbor 90.0.0.6 remote-as 520
```  

Настройка на R23:  
```
R23(config)#router bgp 520
R23(config-router)#bgp router-id 80.0.1.23
R23(config-router)#neighbor 90.0.0.5 remote-as 101
```   

## Настройка iBGP в офисе Москва между маршрутизаторами R14 и R15

Для настройки iBGP между маршрутизаторами R14 и R15 необходимо на каждом прописать соседа.  

Настройка на R14 (на R15 аналогично):  
```
R14(config)#router bgp 1001
R14(config-router)#neighbor 192.168.100.15 remote-as 1001
R14(config-router)#neighbor 192.168.100.15 update-source loopback 0
```  

После настройки на R15 сразу установились соседство и появилась запись в консоли об этом:  
```
R15(config-router)#
*Aug 21 14:46:23.407: %BGP-5-ADJCHANGE: neighbor 192.168.100.14 Up
```  

## Настройка iBGP в провайдере Триада с использованием RR

Для настройки iBGP с помощью route-reflector необходимо выбрать RR-server (им стал R24, потому что через него идёт самый короткий путь между офисами, кроме того, он напрямую соединён с Ламасом, который должен стать приоритетным для офиса Москвы), после чего прописать на нём всех соседей (с помощью peer-group), а на остальных маршрутизаторах прописать в iBGP-соседях RR-server.  

Настройка на R24:  
```
R24(config-router)#neighbor ibgp_clients peer-group
R24(config-router)#neighbor ibgp_clients update-source Loop0
R24(config-router)#neighbor ibgp_clients remote-as 520
R24(config-router)#neighbor ibgp_clients route-reflector-client
R24(config-router)#
R24(config-router)#
R24(config-router)#neighbor 80.0.1.23 peer-group ibgp_clients
R24(config-router)#neighbor 80.0.1.26 peer-group ibgp_clients
```  

Настройка на R23 (на R26 аналогично):  
```
R23(config-router)#neighbor 80.0.1.24 remote-as 520
R23(config-router)#neighbor 80.0.1.24 update-source loop0
```  

При проверке соседей и переданных подсетей на R26 видно, что, не смотря на отсутствие каких-либо других iBGP-соседей, кроме R24, маршрутизатор всё равно получает подсеть офиса Москвы:  
```
R26#sh ip bgp summ
BGP router identifier 80.0.1.26, local AS number 520
BGP table version is 13, main routing table version 13
6 network entries using 888 bytes of memory
8 path entries using 512 bytes of memory
6/4 BGP path/bestpath attribute entries using 816 bytes of memory
1 BGP rrinfo entries using 24 bytes of memory
3 BGP AS-PATH entries using 72 bytes of memory
0 BGP route-map cache entries using 0 bytes of memory
0 BGP filter-list cache entries using 0 bytes of memory
BGP using 2312 total bytes of memory
BGP activity 6/0 prefixes, 13/5 paths, scan interval 60 secs

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
80.0.0.21       4         2042     149     148       13    0    0 02:07:38        2
80.0.1.24       4          520     252     244       13    0    0 03:37:57        6
R26#
R26#
R26#
R26#sh ip bgp
<...>
     Network          Next Hop            Metric LocPrf Weight Path
 r>i 90.0.0.4/30      90.0.0.5                 0    100      0 101 i
 *>i 192.168.2.0/23   90.0.0.9                 0    100      0 301 1001 i
 * i 192.168.6.0/23   80.0.0.17          1541120    100      0 2042 i
 *>                   80.0.0.21          1541120             0 2042 i
 *>i 192.168.100.14/32
                       90.0.0.9                 0    100      0 301 1001 i
 *>i 192.168.100.15/32
                       90.0.0.9                 0    100      0 301 1001 i
 * i 192.168.100.69/32
                       80.0.0.17                0    100      0 2042 i
 *>                   80.0.0.21                0             0 2042 i
```  

## Настройка офиса Москвы так, чтобы приоритетным провайдером стал Ламас

Для настройки приоритетного провайдера необходимо поменять local-prefixes маршрутов от Ламаса на R15 так, чтобы они были больше, чем local-prefix, выставляемый по умолчанию, чтобы при обмене с iBGP-соседями эти маршруты были приоритетней таких же, но полученных другим путём. 

Настройка на R15:  
```
R15(config)#route-map lamas_prefixes
R15(config-route-map)#set local-preference 200
R15(config-route-map)#exit
R15(config)#router bgp 1001
R15(config-router)#neighbor 60.0.0.1 route-map lamas_prefixes in
```  

Проверка работы данной настройки:  
```
R14#sh ip bgp
<...>
     Network          Next Hop            Metric LocPrf Weight Path
 *>i 90.0.0.4/30      60.0.0.1                 0    200      0 301 101 i
 *                    70.0.0.1                 0             0 101 i
 *>  192.168.100.14/32
                       0.0.0.0                  0         32768 i
 r>i 192.168.100.15/32
                       192.168.100.15           0    100      0 i
 *   192.168.100.69/32
                       70.0.0.1                               0 101 520 2042 i
 *>i                  60.0.0.1                 0    200      0 301 520 2042 i
```  

Одни и те же маршруты, но имеющие next-hop через Ламас, а не Киторн, приоритетнее за счёт большего local-preference.  

## Настройка офиса С.-Петербург так, чтобы трафик до любого офиса распределялся по двум линкам одновременно

Для распределения трафика по двум линкам одновременно необходимо использовать команду *maximum-path* в настройках BGP пограничного маршрутизатора офиса С.-Петербурга.

Настройка на R18:  
```
R18(config)#router bgp 2042
R18(config-router)#maximum-paths 2
```  

После этого при просмотре маршрутов, приходящих извне, видно, что у них 2 next-hop:  
```
R18#sh ip bgp
<...>
     Network          Next Hop            Metric LocPrf Weight Path
 *m  90.0.0.4/30      80.0.0.22                              0 520 101 i
 *>                   80.0.0.18                              0 520 101 i
 *m  192.168.100.14/32
                       80.0.0.22                              0 520 301 1001 i
 *>                   80.0.0.18                              0 520 301 1001 i
 *m  192.168.100.15/32
                       80.0.0.22                              0 520 301 1001 i
 *>                   80.0.0.18                              0 520 301 1001 i
 *>  192.168.100.69/32
                       0.0.0.0                  0         32768 i
R18#
R18#
R18#
R18#sh ip rou
<...>
      90.0.0.0/30 is subnetted, 1 subnets
B        90.0.0.4 [20/0] via 80.0.0.22, 02:26:13
                  [20/0] via 80.0.0.18, 02:26:13
D     192.168.6.0/23 [90/1541120] via 10.0.0.89, 01:53:06, Ethernet0/0
                     [90/1541120] via 10.0.0.73, 01:53:06, Ethernet0/1
      192.168.100.0/24 is variably subnetted, 4 subnets, 2 masks
B        192.168.100.14/32 [20/0] via 80.0.0.22, 02:26:13
                           [20/0] via 80.0.0.18, 02:26:13
B        192.168.100.15/32 [20/0] via 80.0.0.22, 02:26:13
                           [20/0] via 80.0.0.18, 02:26:13
C        192.168.100.69/32 is directly connected, Loopback0
```  

## Настройка IP-связности всех сетей в лабораторной работе

Для настройки IP-связности сетей офисов Москвы и Питера необходимо было донастроить маршрутизацию - прописать default-gateway в офисах и анонсировать рабочие сети в BGP.

Настройки на R14 (на R15 аналогично):  
```
R14(config)#ip route 192.168.2.0 255.255.254.0 Null 0
R14(config)#
R14(config)#
R14(config)#router bgp 1001
R14(config-router)#network 192.168.2.0 mask 255.255.254.0
```  

Настройки на R15:  
```
R15(config)#ip route 0.0.0.0 0.0.0.0 60.0.0.1 254
```  

Настройки на R18:  
```
R18(config)#router bgp 2042
R18(config-router)#network 192.168.6.0 mask 255.255.254.0
R18(config)#
R18(config)#
R18(config)#router eigrp SPB
R18(config-router)#address-family ipv4 autonomous-system 13
R18(config-router-af)#af-interface e0/1
R18(config-router-af-interface)#summary-address 0.0.0.0/0
R18(config-router-af-interface)#exi
R18(config-router-af)#
R18(config-router-af)#
R18(config-router-af)#af-interface e0/0        
R18(config-router-af-interface)#summary-address 0.0.0.0/0
```  

После этого для проверки связности произведён пинг с компьютера из рабочей сети Москвы до компьютера в Питере и обратно:  
```
msk_user_pc> ping 192.168.6.2

84 bytes from 192.168.6.2 icmp_seq=1 ttl=55 time=3.788 ms
84 bytes from 192.168.6.2 icmp_seq=2 ttl=55 time=3.221 ms
84 bytes from 192.168.6.2 icmp_seq=3 ttl=55 time=3.407 ms
84 bytes from 192.168.6.2 icmp_seq=4 ttl=55 time=2.283 ms
84 bytes from 192.168.6.2 icmp_seq=5 ttl=55 time=3.159 ms

msk_user_pc>
```  
```
spb_user_pc> ping 192.168.2.2

84 bytes from 192.168.2.2 icmp_seq=1 ttl=56 time=3.147 ms
84 bytes from 192.168.2.2 icmp_seq=2 ttl=56 time=6.304 ms
84 bytes from 192.168.2.2 icmp_seq=3 ttl=56 time=4.635 ms
84 bytes from 192.168.2.2 icmp_seq=4 ttl=56 time=2.983 ms
84 bytes from 192.168.2.2 icmp_seq=5 ttl=56 time=3.365 ms

spb_user_pc> 
```  

Можно сделать вывод, что связность присутствует (если я правильно поняла идею связности).