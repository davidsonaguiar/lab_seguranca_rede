\# L2 — Bloqueio por endereço MAC



\## Cenário



O `pc2` (10.0.1.11) foi identificado como dispositivo comprometido. O firewall deve bloquear o tráfego vindo do MAC dele, sem afetar o `pc1`.



| Máquina | IP | MAC |

|---|---|---|

| pc1 | 10.0.1.10 | `66:51:fe:5a:86:5f` |

| pc2 | 10.0.1.11 | `86:a2:e7:f2:25:c6` |

| fw (eth1) | 10.0.1.1 | `76:23:1e:4b:4c:72` |



> Os MACs são gerados pelo Docker e mudam a cada `kathara lstart`. É preciso descobri-los de novo com `ip link show eth0` ao subir o laboratório.



\## 1. Antes da regra



Ping do `pc2` e do `pc1` para o `r0` (198.51.100.2). Os dois funcionam.



Ping do `pc2`:



```

root@pc2:/# ping -c 3 198.51.100.2

64 bytes from 198.51.100.2: icmp\_seq=1 ttl=63 time=1.55 ms

64 bytes from 198.51.100.2: icmp\_seq=2 ttl=63 time=0.697 ms

64 bytes from 198.51.100.2: icmp\_seq=3 ttl=63 time=0.700 ms

3 packets transmitted, 3 received, 0% packet loss

```



Captura no `fw` (`tcpdump -i eth1 -n -e icmp`), mostrando os MACs de origem:



```

86:a2:e7:f2:25:c6 > 76:23:1e:4b:4c:72 ... 10.0.1.11 > 198.51.100.2: ICMP echo request, seq 1

76:23:1e:4b:4c:72 > 86:a2:e7:f2:25:c6 ... 198.51.100.2 > 10.0.1.11: ICMP echo reply, seq 1

66:51:fe:5a:86:5f > 76:23:1e:4b:4c:72 ... 10.0.1.10 > 198.51.100.2: ICMP echo request, seq 1

76:23:1e:4b:4c:72 > 66:51:fe:5a:86:5f ... 198.51.100.2 > 10.0.1.10: ICMP echo reply, seq 1

```



Resultado: `pc1` e `pc2` recebem resposta (3 de 3, sem perda).



\## 2. Regra implementada



No `fw`:



```bash

iptables -I FORWARD 1 -i eth1 -m mac --mac-source 86:a2:e7:f2:25:c6 -j DROP

```



A regra é inserida na posição 1 para ser avaliada antes das regras de `ACCEPT`.



\## 3. Depois da regra



Ping do `pc2`:



```

root@pc2:/# ping -c 3 198.51.100.2

PING 198.51.100.2 (198.51.100.2) 56(84) bytes of data.

3 packets transmitted, 0 received, 100% packet loss, time 2049ms

```



Ping do `pc1`: continua funcionando, com 3 de 3 respostas e sem perda.



Contadores no `fw` (`iptables -L FORWARD -n -v --line-numbers`), zerados antes do teste com `iptables -Z FORWARD`:



```

num   pkts bytes target  prot opt in    out    source     destination

1        3   252 DROP    0    --  eth1  \*      0.0.0.0/0  0.0.0.0/0   MAC 86:a2:e7:f2:25:c6

2        5   420 ACCEPT  0    --  \*     \*      0.0.0.0/0  0.0.0.0/0   ctstate RELATED,ESTABLISHED

3        1    84 ACCEPT  0    --  eth1  eth0   0.0.0.0/0  0.0.0.0/0

```



A linha 1 descartou 3 pacotes de 84 bytes (252 bytes), que correspondem aos 3 pings do `pc2`. O tráfego do `pc1` passou pelas linhas 2 e 3.



\## 4. Explicação



A regra compara o MAC de origem do quadro Ethernet que chega na `eth1`. Como o `pc2` é a única máquina com esse MAC, só o tráfego dele é descartado, e o `pc1` continua normal.



\## 5. Investigação



\*\*O endereço MAC acompanha um pacote durante todo o seu percurso pela Internet?\*\*

Não. O MAC só tem significado dentro de um mesmo segmento de rede (camada 2). A cada roteador, o cabeçalho Ethernet é descartado e refeito com os MACs do próximo salto. Na captura acima, o `pc2` envia os pacotes ao MAC do `fw` (seu gateway), e não ao do destino final.



\*\*Em quais condições o firewall consegue enxergar o MAC original do `pc2`?\*\*

Quando o `pc2` está no mesmo segmento que a interface do firewall, como aqui, onde ele está na LAN, ligado diretamente à `eth1`. Se houvesse outro roteador entre eles, o firewall veria o MAC desse roteador, e a regra bloquearia todas as máquinas atrás dele.



\*\*Limitação:\*\* quem controla a máquina pode trocar o MAC (MAC spoofing), então esse bloqueio, sozinho, é fraco contra um dispositivo realmente comprometido.

