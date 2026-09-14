# Firewall + DMZ: Segurança de Perímetro e Defense in Depth

**Objetivo**

Implementar no Kathará a topologia apresentada na figura desta Task e utilizá-la como um pequeno Cyber Range (laboratório experimental) para experimentar conceitos fundamentais de Cibersegurança:

- Segurança de Perímetro;
- Firewall e filtragem de tráfego;
- DMZ e segmentação de redes;
- Default Deny e menor privilégio;
- controles em diferentes camadas da pilha TCP/IP;
- Defense in Depth (Defesa em Profundidade).

> Utilize exatamente a topologia e o plano de endereçamento apresentados na figura.

![alt text](4a3e1d1d-1f97-4fe2-99b5-95c9d8f77152.jpg)

## [1. Implemente a topologia](1_implemente_a_topologia/README.md)

Reproduza a topologia no Kathará, configurando os dispositivos e redes apresentados na figura.

Configure:

- ``r0`` — roteador de borda e acesso à Internet;
- ``fw`` — firewall/middlebox;
- ``pc1`` e ``pc2`` — estações da LAN;
- ``web`` — servidor Web na DMZ;
- ``dns`` — servidor DNS na DMZ;
- rede de gerenciamento — opcional.

Inicialmente, **não configure regras restritivas no firewall**. Configure endereçamento, roteamento e acesso à Internet.

### Validação inicial

Demonstre que:

- LAN → Internet funciona;
- LAN → DMZ funciona;
- Web e DNS estão acessíveis;
- o firewall encaminha corretamente os pacotes entre as redes.

Essa será a **baseline** para os próximos experimentos.

## [2. Implemente a Segurança de Perímetro](2_implemente_a_seguranca_de_perimetro/README.md)

Transforme ``fw`` em um **Firewall de Perímetro**, utilizando ``iptables`` ou ``nftables``.

Adote como princípio:

> Bloquear por padrão e permitir explicitamente apenas as comunicações necessárias.

Implemente, no mínimo, a seguinte política:

| Comunicação | Política |
| --- | --- | --- |
| LAN → Internet | ✅ Permitir |
| LAN → Web/DNS da DMZ | ✅ Permitir |
| Internet → Web da DMZ | ✅ Permitir |
| Internet → LAN | ❌ Bloquear |
| DMZ → LAN (novas conexões) | ❌ Bloquear |
|Respostas de conexões permitidas | ✅ Permitir |

Sempre que apropriado, utilize filtragem stateful para diferenciar novas conexões das respostas pertencentes a conexões já estabelecidas.

## [3. Experimente controles em diferentes camadas](3_experimente_controles_em_diferentes_camadas/README.md)

Agora vamos investigar como decisões de segurança podem utilizar informações provenientes de diferentes camadas da comunicação.

Para cada experimento, sigam o ciclo:

**Gere o tráfego → Observe → Aplique a regra → Teste novamente → Explique o resultado.**

Utilizem tcpdump ou Wireshark sempre que necessário para observar os pacotes.

---

### 🔗 L2 — Enlace: bloquear um dispositivo

Considere que ``pc2`` foi identificado como um dispositivo comprometido e está gerando **tráfego malicioso**.

Configure o firewall para **bloquear o tráfego proveniente do endereço MAC de ``pc2``**.

Compare o comportamento de ``pc1`` e ``pc2`` antes e depois da regra.
**Investigue**: o endereço MAC acompanha um pacote durante todo o seu percurso pela Internet? Em quais condições o firewall consegue enxergar o MAC original de ``pc2``?

---

### 🌐 L3 — Rede: ICMP e bloqueio de destinos

Realize dois experimentos:

**A. ICMP**: bloqueie tráfego ICMP entre duas redes. Utilize ``ping`` e ``tcpdump`` para demonstrar o efeito da regra.

**B. Destino IP**: considere que determinado endereço IP representa um destino externo que a organização decidiu proibir. Bloqueie o acesso da LAN a esse endereço.

Não é necessário utilizar um site realmente malicioso ou impróprio: um servidor controlado pelo laboratório pode representar o destino proibido.

**Investigue**: bloquear o endereço IP é uma boa solução para impedir o acesso a determinado site?

Considere que um domínio pode possuir vários IPs, um mesmo IP pode hospedar diversos sites e serviços podem utilizar CDNs ou alterar seus endereços.

---

### 🚪 L4 — Transporte: bloqueio de serviços

Considere que a organização possui uma política que proíbe determinados serviços **P2P**, como BitTorrent.

Utilize protocolos e **portas TCP/UDP** para implementar uma política que represente o bloqueio desse tipo de serviço.

Gere tráfego de teste nas portas escolhidas e demonstre o bloqueio.

**Investigue**: bloquear portas é suficiente para garantir que uma aplicação como BitTorrent não funcione?

Considere a possibilidade de aplicações utilizarem portas alternativas ou dinâmicas.

---

### 🧩 L7 — Aplicação: investigue controles mais inteligentes

Até aqui utilizamos informações progressivamente mais específicas:
**MAC → IP → protocolo/porta**

Na camada de aplicação podemos tomar decisões com maior conhecimento sobre **qual aplicação está sendo utilizada e o que ela está fazendo**.

Investiguem mecanismos que permitam, por exemplo:

- bloquear um domínio específico;
- bloquear categorias de sites;
- permitir /public e bloquear /admin;
- identificar aplicações independentemente da porta utilizada;
- controlar determinados recursos ou conteúdos de uma aplicação.

Pesquisem tecnologias como **DNS Filtering, Proxy, WAF, Application Firewall e NGFW**.

> Não é necessário implementar L7 nesta Task.

Cada grupo deverá **escolher uma possibilidade de controle em L7**, explicar brevemente como ela funciona e **trazê-la para discussão na próxima aula**.

---

## 4. Defense in Depth

Observe novamente a arquitetura completa.

Considere o seguinte cenário:

> O servidor Web da DMZ foi comprometido. Isso significa que o atacante agora consegue acessar diretamente pc1 e pc2?

Identifique quais controles da arquitetura ainda poderiam limitar o acesso à LAN.

Discuta como **Firewall + DMZ + segmentação + regras de filtragem + controles nos próprios serviços** formam diferentes camadas de proteção.

Esse é o princípio de Defense in Depth:

> Não depender de uma única barreira de segurança. Se uma camada falhar, outras camadas ainda devem ajudar a limitar o ataque.