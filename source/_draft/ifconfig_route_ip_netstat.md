[net-tools](https://github.com/ecki/net-tools) 与 [iproute2](https://github.com/shemminger/iproute2)

[net-tools](https://net-tools.sourceforge.io/)包括: `ifconfig`、`ipmaddr`、`route`、`arp`、`rarp`、`netstat`等命令。从[bsd](https://github.com/openbsd/src)项目中出来的。

而[iproute2](https://en.wikipedia.org/wiki/Iproute2)是更现代化的工具，包括`ip`、`ss`等众多命令，特别是`ip`命令通过子命令将很多功能整合到了一起。

|                     Legacy utility                     |   Replacement command   |                Note                 |
| :----------------------------------------------------: | :---------------------: | :---------------------------------: |
|   [ifconfig](https://en.wikipedia.org/wiki/Ifconfig)   | ip addr, ip link, ip -s |   Address and link configuration    |
| [route](https://en.wikipedia.org/wiki/Route_(command)) |        ip route         |           Routing tables            |
|                          arp                           |        ip neigh         |              Neighbors              |
|                        iptunnel                        |        ip tunnel        |               Tunnels               |
|                    nameif, ifrename                    |    ip link set name     |      Rename network interfaces      |
|                        ipmaddr                         |        ip maddr         |              Multicast              |
|    [netstat](https://en.wikipedia.org/wiki/Netstat)    |   ip -s, ss, ip route   | Show various networking statistics  |
|                         brctl                          |         bridge          | Handle bridge addresses and devices |

