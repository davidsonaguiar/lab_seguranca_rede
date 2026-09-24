Aqui está um guia de testes estruturado em **Markdown**, pronto para você utilizar como roteiro de validação do laboratório e anexar diretamente na documentação do seu relatório.

---

# Roteiro de Validação e Testes do Cyber Range

Este documento contém a sequência completa de comandos e explicações para validar todas as etapas de segurança da topologia.

---

## 1. Testes de Baseline e Conectividade Global

Garante que o roteamento e a infraestrutura básica de rede funcionam antes de validar as políticas restritivas do firewall.

### 1.1. Teste de Acesso Web Interno (LAN → DMZ)

Acessa o servidor Web da DMZ a partir de um host da rede interna.

* **Onde executar:** `pc1`
* **Comando:**
```bash
curl -i http://10.0.2.10

```


* **Resultado Esperado:** Retorno HTTP `200 OK` exibindo o HTML do servidor Web.

---

### 1.2. Teste de Resolução de Nomes (LAN → DNS DMZ)

Consulta a resolução de nomes da zona `portal.dmz` diretamente no servidor DNS da DMZ, utilizando apenas o tipo A (IPv4) para evitar falhas com IPv6.

* **Onde executar:** `pc1`
* **Comando:**
```bash
host portal.dmz 10.0.2.11

```


* **Resultado Esperado:**
```text
portal.dmz has address 10.0.2.10

```



---

### 1.3. Teste de Acesso Web por Domínio (LAN → DMZ via Nome)

Valida a integração da resolução DNS com a navegação HTTP.

* **Onde executar:** `pc1`
* **Comando:**
```bash
curl -i http://portal.dmz

```


* **Resultado Esperado:** Retorno HTTP `200 OK`.

---

### 1.4. Teste de Saída para a Internet (LAN → WAN)

Garante que a estação interna alcança redes externas (simuladas ou via roteador de borda).

* **Onde executar:** `pc1`
* **Comando:**
```bash
ping -c 3 8.8.8.8

```


* **Resultado Esperado:** `0% packet loss` (respostas ICMP com sucesso).

---

## 2. Testes da Segurança de Perímetro (Firewall Stateful & Default Deny)

Valida as políticas aplicadas no `fw.startup` baseadas no princípio do bloqueio por padrão e inspeção com `conntrack`.

### 2.1. Permissão: Internet → Web da DMZ

Simula um usuário externo na Internet acessando a aplicação pública.

* **Onde executar:** `r0`
* **Comando:**
```bash
curl -i http://10.0.2.10

```


* **Resultado Esperado:** Retorno HTTP `200 OK`.

---

### 2.2. Bloqueio: Internet → LAN

Garante que pacotes vindos da WAN não consigam atingir diretamente a rede interna de máquinas.

* **Onde executar:** `r0`
* **Comando:**
```bash
ping -c 3 10.0.1.10

```


* **Resultado Esperado:** `100% packet loss` (descartado pela política `FORWARD DROP`).

---

### 2.3. Bloqueio: DMZ → LAN (Conexão Iniciada pela DMZ)

Verifica o isolamento da DMZ, garantindo que um servidor comprometido na DMZ não consiga varrer nem iniciar conexões contra os computadores da LAN.

* **Onde executar:** `web`
* **Comando:**
```bash
ping -c 3 10.0.1.10

```


* **Resultado Esperado:** `100% packet loss` / Timeout.

---

### 2.4. Inspeção dos Contadores do Firewall

Inspeciona as estatísticas das regras no firewall para evidenciar a atuação do filtro de pacotes.

* **Onde executar:** `fw`
* **Comando:**
```bash
iptables -L FORWARD -v -n --line-numbers

```


* **Resultado Esperado:** Incremento no contador de pacotes da regra `ESTABLISHED,RELATED` e da política de `policy DROP`.

---

## 3. Testes de Bloqueio por Camada (L2, L3, L4 e L7)

> **Nota de Execução:** Execute **um teste por vez**. Aplique a regra no `fw`, faça o teste e remova a regra com `iptables -D FORWARD 1` antes de seguir para a próxima camada.

---

### 3.1. Camada 2 (Enlace) — Bloqueio por Endereço MAC

Bloqueia todo o tráfego gerado pela placa de rede física de uma estação específica (`pc2`).

#### 1. Capturar o MAC do `pc2`

* **Onde executar:** `pc2`
* **Comando:**
```bash
ip link show eth0

```


* **Ação:** Anote o endereço após `link/ether` (exemplo: `02:42:0a:00:01:0b`).

#### 2. Aplicar a Regra

* **Onde executar:** `fw`
* **Comando:**
```bash
iptables -I FORWARD 1 -m mac --mac-source <MAC_DO_PC2> -j DROP

```



#### 3. Executar Validação

* **No `pc2` (Bloqueado):**
```bash
curl -m 3 http://10.0.2.10

```


*(Resultado: Timeout)*
* **No `pc1` (Liberado):**
```bash
curl http://10.0.2.10

```


*(Resultado: HTTP 200 OK)*

#### 4. Limpar Regra

* **Onde executar:** `fw`
* **Comando:** `iptables -D FORWARD 1`

---

### 3.2. Camada 3 (Rede) — Bloqueio por IP de Origem

Bloqueia todo o tráfego IP originado da estação `pc2` (`10.0.1.11`).

#### 1. Aplicar a Regra

* **Onde executar:** `fw`
* **Comando:**
```bash
iptables -I FORWARD 1 -s 10.0.1.11 -j DROP

```



#### 2. Executar Validação

* **No `pc2` (Bloqueado):**
```bash
ping -c 3 10.0.2.10

```


*(Resultado: 100% packet loss)*
* **No `pc1` (Liberado):**
```bash
ping -c 3 10.0.2.10

```


*(Resultado: 0% packet loss)*

#### 3. Limpar Regra

* **Onde executar:** `fw`
* **Comando:** `iptables -D FORWARD 1`

---

### 3.3. Camada 4 (Transporte) — Bloqueio por Porta de Serviço (HTTP 80)

Bloqueia requisições do protocolo TCP destinadas à porta `80` vindas da LAN, mantendo o tráfego ICMP (Ping) liberado.

#### 1. Aplicar a Regra

* **Onde executar:** `fw`
* **Comando:**
```bash
iptables -I FORWARD 1 -i eth1 -p tcp --dport 80 -j DROP

```



#### 2. Executar Validação

* **No `pc1` — Teste HTTP (Bloqueado):**
```bash
curl -m 3 http://10.0.2.10

```


*(Resultado: Timeout — Camada 4 filtrada)*
* **No `pc1` — Teste Ping (Liberado):**
```bash
ping -c 3 10.0.2.10

```


*(Resultado: 0% packet loss — Camada 3 operante)*

#### 3. Limpar Regra

* **Onde executar:** `fw`
* **Comando:** `iptables -D FORWARD 1`

---

### 3.4. Camada 7 (Aplicação) — Bloqueio por Inspeção de Payload (String)

Inspeciona o conteúdo dos pacotes HTTP e descarta chamadas que contenham o nome do domínio `"portal.dmz"` no cabeçalho `Host`.

#### 1. Aplicar a Regra

* **Onde executar:** `fw`
* **Comando:**
```bash
iptables -I FORWARD 1 -p tcp --dport 80 -m string --string "portal.dmz" --algo bm -j DROP

```



#### 2. Executar Validação

* **No `pc1` — Acesso via Domínio (Bloqueado):**
```bash
curl -m 3 -i http://portal.dmz

```


*(Resultado: Timeout — String detectada no payload HTTP)*
* **No `pc1` — Acesso via IP Direto (Liberado):**
```bash
curl -i http://10.0.2.10

```


*(Resultado: HTTP 200 OK — O cabeçalho por IP não carrega o nome do domínio)*

#### 3. Limpar Regra

* **Onde executar:** `fw`
* **Comando:** `iptables -D FORWARD 1`

---

## Resumo dos Resultados Esperados

| Etapa | Teste Executado | Origem | Destino | Status Esperado |
| --- | --- | --- | --- | --- |
| **1. Baseline** | HTTP / DNS / Ping | `pc1` | DMZ / Internet | ✅ Permitido |
| **2. Perímetro** | HTTP Web DMZ | `r0` (WAN) | `10.0.2.10` | ✅ Permitido |
| **2. Perímetro** | Ping Direto LAN | `r0` (WAN) | `10.0.1.10` | ❌ Bloqueado |
| **2. Perímetro** | Ping Iniciado DMZ | `web` (DMZ) | `10.0.1.10` | ❌ Bloqueado |
| **3.1. L2 (MAC)** | MAC Filtro `pc2` | `pc2` vs `pc1` | DMZ Web | ❌ `pc2` / ✅ `pc1` |
| **3.2. L3 (IP)** | IP `10.0.1.11` | `pc2` vs `pc1` | DMZ Web | ❌ `pc2` / ✅ `pc1` |
| **3.3. L4 (Porta)** | HTTP Port 80 vs Ping | `pc1` | DMZ Web | ❌ HTTP / ✅ Ping |
| **3.4. L7 (String)** | `portal.dmz` vs `IP` | `pc1` | DMZ Web | ❌ Domínio / ✅ IP |