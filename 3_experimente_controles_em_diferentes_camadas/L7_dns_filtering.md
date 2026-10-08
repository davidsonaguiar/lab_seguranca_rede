\# L7 — DNS Filtering



\## O que é



O DNS Filtering é um controle de camada de aplicação (L7) que bloqueia o acesso a um site pelo \*\*nome\*\* (domínio), e não pelo endereço IP.



\## Como funciona



1\. O usuário tenta acessar `site-proibido.com`.

2\. Antes de conectar, o computador pergunta ao servidor DNS qual é o IP desse nome.

3\. O servidor DNS da organização consulta uma lista de domínios bloqueados.

4\. Se o domínio estiver na lista, o servidor \*\*não devolve o IP verdadeiro\*\*. Ele responde com um erro de domínio inexistente (`NXDOMAIN`) ou com o IP de uma página de aviso (sinkhole).

5\. Sem o IP, o navegador não consegue conectar, e o acesso é barrado logo no início, antes de qualquer conexão TCP ou TLS.



\## Vantagens



\- É simples de configurar e tem baixo impacto no desempenho, pois não inspeciona o tráfego.

\- Resolve a limitação do bloqueio por IP (L3), que falha com CDNs, IPs compartilhados e domínios com vários endereços. No DNS, bloqueamos o \*\*nome\*\*, que é o que o usuário realmente quer acessar.

\- Na topologia deste laboratório, poderia ser implementado no servidor `dns` da DMZ, por exemplo com o `dnsmasq`, usando uma linha como `address=/dominio-bloqueado.com/0.0.0.0`.



\## Limitações



\- \*\*Só enxerga o domínio.\*\* Não é possível bloquear `site.com/admin` e liberar `site.com/public`: ou bloqueia o site inteiro, ou nada.

\- \*\*Pode ser contornado:\*\*

&#x20; - alterando manualmente o servidor DNS da máquina;

&#x20; - usando DNS criptografado (DoH ou DoT), que impede o servidor da organização de ver a consulta;

&#x20; - usando VPN;

&#x20; - acessando o destino diretamente pelo IP.

\- \*\*Não inspeciona o conteúdo.\*\* Se o domínio é liberado, o DNS Filtering não vê o que trafega depois da resolução.



\## Como reduzir o contorno



O firewall pode obrigar a LAN a usar apenas o DNS da organização: libera a porta 53 somente para o servidor `dns` e bloqueia as portas 53 e 853 (DoT) para qualquer outro destino. O DoH continua passando, pois usa a porta 443 como qualquer tráfego HTTPS. Para tratá-lo, é necessário um proxy ou NGFW com inspeção TLS.



\## Controles mais granulares



Para decisões mais finas (por exemplo, permitir `/public` e bloquear `/admin`), seriam necessários mecanismos como Proxy, WAF ou NGFW, que enxergam a URL e o conteúdo da aplicação.

