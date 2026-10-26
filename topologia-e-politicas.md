# Topologia e politicas do laboratorio

Este documento descreve a topologia Kathara e as politicas efetivamente carregadas no cenario `controles_diferentes_camadas`. Os testes passo a passo e as evidencias estao no [README principal](README.md).

## Componentes e segmentos

O arquivo `controles_diferentes_camadas/lab.conf` conecta os nos a quatro redes:

| Zona | Nos/interfaces | Funcao |
|---|---|---|
| LAN | `pc1`, `pc2` e interface `eth1` de `firewall` | Estacoes internas |
| DMZ | `web`, `dns` e interface `eth2` de `firewall` | Servicos publicados/compartilhados |
| WAN | `roteador_borda` e interface `eth0` de `firewall` | Saida do perimetro para a rede externa |
| Gerenciamento | `adm` e interface `eth3` de `firewall` | Segmento separado para administracao |

No enunciado, `firewall` corresponde ao `fw` e `roteador_borda` ao `r0`. A rede de gerenciamento e adicional/opcional. O roteador de borda tem uma interface conectada a `wan` e uma interface externa bridged, usada para a conectividade com a rede externa.

## Enderecamento e roteamento

| No | Interface | Endereco | Gateway/rota relevante |
|---|---|---|---|
| `pc1` | `eth0` | `10.0.1.10/24` | Default via `10.0.1.1` |
| `pc2` | `eth0` | `10.0.1.11/24` | Default via `10.0.1.1` |
| `firewall` | `eth0` (WAN) | `198.51.100.2/30` | Default via `198.51.100.1` |
| `firewall` | `eth1` (LAN) | `10.0.1.1/24` | Rede diretamente conectada |
| `firewall` | `eth2` (DMZ) | `10.0.2.1/24` | Rede diretamente conectada |
| `firewall` | `eth3` (gerenciamento) | `10.0.3.1/24` | Rede diretamente conectada |
| `web` | `eth0` | `10.0.2.10/24` | Default via `10.0.2.1` |
| `dns` | `eth0` | `10.0.2.11/24` | Default via `10.0.2.1` |
| `adm` | `eth0` | `10.0.3.10/24` | Default via `10.0.3.1` |
| `roteador_borda` | `eth0` (WAN interna) | `198.51.100.1/30` | Rota `10.0.0.0/16` via `198.51.100.2` |

O `web` executa um servidor HTTP na porta TCP/80. O `dns` executa `dnsmasq` e fornece resolucao DNS. O roteador de borda aplica NAT/MASQUERADE na interface externa. As estacoes apontam primeiro para `10.0.2.11` como DNS organizacional; a configuracao tambem lista `8.8.8.8` como alternativa.

## Caminhos de comunicacao

- **LAN → Internet:** `pc1`/`pc2` enviam o trafego ao gateway `10.0.1.1`; o firewall o encaminha para a WAN (`eth0`), o roteador de borda faz NAT na interface externa e envia o trafego adiante.
- **LAN → DMZ:** o trafego passa pelo firewall entre `eth1` e `eth2`. Na mesma LAN, `pc1` e `pc2` tambem podem trocar trafego diretamente, sem atravessar o firewall.
- **Internet → DMZ:** o trafego destinado a servicos publicados passa pelo roteador de borda e pelo firewall antes de chegar a DMZ.
- **DMZ → LAN:** nao ha regra permitindo novas conexoes iniciadas na DMZ para a LAN. Com a politica `FORWARD DROP`, esses novos fluxos sao descartados. Respostas pertencentes a fluxos permitidos sao tratadas pela regra stateful.
- **Gerenciamento:** `adm` esta em uma sub-rede distinta. A configuracao atual nao adiciona excecao `INPUT` para administracao do proprio firewall, portanto essa segmentacao nao significa que o acesso administrativo ao firewall ja esteja habilitado.

## Politicas efetivas do firewall

As regras estao em `controles_diferentes_camadas/firewall.startup`. No inicio, sao definidas as politicas padrao `DROP` para `INPUT`, `OUTPUT` e `FORWARD`; depois, as cadeias sao limpas e as permissoes de encaminhamento sao adicionadas.

| Cadeia/direcao | Regra efetiva | Consequencia |
|---|---|---|
| `INPUT` | Politica `DROP`; sem excecoes explicitas | Pacotes destinados ao proprio firewall sao bloqueados |
| `OUTPUT` | Politica `DROP`; sem excecoes explicitas | Conexoes iniciadas pelo proprio firewall sao bloqueadas |
| `FORWARD` | Politica `DROP` | Trafego roteado sem regra de permissao e descartado |
| `FORWARD` | Permite `ESTABLISHED,RELATED` | Permite respostas e trafego relacionado a conexoes encaminhadas permitidas |
| LAN (`eth1`) → WAN (`eth0`) | Permite encaminhamento em geral | LAN pode iniciar conexoes para a WAN sem restricao por porta nesta regra |
| LAN (`eth1`) → DMZ (`eth2`) | Permite ICMP | Hosts da LAN podem iniciar ICMP para a DMZ |
| LAN → servidor Web | Permite TCP/80 e TCP/443 para `10.0.2.10` | Acesso HTTP/HTTPS permitido pela regra; o servico atual documentado no servidor e somente HTTP |
| LAN → servidor DNS | Permite UDP/53 e TCP/53 para `10.0.2.11` | Consultas DNS ao servidor da DMZ permitidas |
| WAN (`eth0`) → DMZ (`eth2`) | Permite ICMP para a DMZ | A regra nao restringe ICMP a um endereco especifico da DMZ |
| WAN → servidor Web | Permite TCP/80 e TCP/443 para `10.0.2.10` | Acesso externo ao Web permitido pela regra; HTTPS ainda nao e fornecido pelo servidor atual |
| WAN → servidor DNS | Permite UDP/53 e TCP/53 para `10.0.2.11` | Consultas DNS iniciadas pela WAN sao permitidas; isso expoe o DNS e nao e necessario para a matriz minima da atividade |
| DMZ → LAN | Nenhuma permissao para novas conexoes | Bloqueado pelo `FORWARD DROP`; respostas de conexoes ja permitidas continuam sujeitas ao estado `ESTABLISHED,RELATED` |
| WAN → LAN | Nenhuma permissao para novas conexoes | Bloqueado pelo `FORWARD DROP` |

### Leitura da politica

A politica padrao das cadeias e `DROP`, mas isso nao significa que toda a politica siga estritamente o menor privilegio: a regra LAN → WAN permite qualquer protocolo/porta, e ha permissoes de ICMP e DNS da WAN para a DMZ. A regra HTTP/HTTPS da WAN para o DNS, em particular, nao e necessaria para publicar o servidor Web. Essas permissoes estao documentadas aqui para distinguir o comportamento real do laboratorio de uma politica recomendada ou ideal.

Os bloqueios L2 por MAC, L3 por ICMP/IP, L4 por portas e L7 por DNS descritos no README principal sao experimentos adicionados manualmente. Nao estao carregados por `firewall.startup`/`dns.startup` como politica permanente e precisam ser aplicados durante a demonstracao (ou adicionados aos arquivos de inicializacao para persistirem).

## Relacao com Defense in Depth

A separacao LAN/DMZ/WAN limita quais caminhos de rede existem entre estacoes internas, servicos publicados e redes externas. O firewall controla o trafego que cruza essas zonas, incluindo o bloqueio de novas conexoes DMZ → LAN. O roteador de borda e NAT oferecem outra etapa no caminho para a rede externa. Os experimentos acrescentam decisao por MAC, IP, protocolo/porta e nome DNS. Finalmente, controles locais dos servicos — atualizacao, autenticacao, permissoes e configuracao segura — continuam necessarios: segmentacao limita o movimento lateral, mas nao corrige uma vulnerabilidade no proprio servidor nem impede ataques contra servicos que foram explicitamente publicados.
