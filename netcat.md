# netcat (nc) — troubleshooting de rede (Linux)

**Objetivo:** documentação curta e prática para **diagnosticar problemas de comunicação** (TCP/UDP), validar **portas/serviços**, simular **cliente/servidor** e fazer testes rápidos em redes Linux.

---

## Para que serve?
O **netcat (`nc`)** é uma ferramenta comum em Linux que ajuda a **entender problemas de rede** e **diagnosticar falhas de comunicação** entre hosts/serviços.

Na prática, ele é útil para:
- Validar se **uma porta** está acessível (firewall/rota/ACL/NAT).
- Confirmar se um **serviço** está “escutando” e respondendo.
- Simular **cliente** e **servidor** rapidamente.
- Fazer testes simples de **TCP/UDP** sem depender do aplicativo original.

---

## O que é netcat
`netcat` (comando `nc`) é um utilitário que **abre conexões TCP/UDP**, **escuta portas** e **envia/recebe dados**. Ele é útil para **isolar se o problema é rede/porta/firewall** ou **aplicação**.

---

## Instalação
Debian/Ubuntu:
```bash
sudo apt update
sudo apt install netcat-openbsd
```

RHEL/Fedora:
```bash
sudo dnf install nmap-ncat
```

Verificar ajuda:
```bash
nc -h
```

---

## Padrões de erro que você precisa reconhecer
- **succeeded / open**: conectou → rede/porta ok (o problema pode ser app, TLS, credencial, etc.).
- **Connection refused**: chegou no host, mas **ninguém está escutando** naquela porta (ou há reject ativo).
- **timed out**: geralmente **bloqueio no caminho** (firewall/ACL/rota/ISP) ou host sem resposta.

---

## Playbook curto para incidentes (netcat-first)
1) **DNS resolve?**
```bash
getent hosts HOST
```

2) **Porta TCP abre?**
```bash
nc -vz -w2 HOST PORTA
```

3) **Se for voz/vídeo (UDP):** teste perda com o modo UDP (seção “Perda UDP”).

4) **Porta abre, mas app falha:** valide resposta mínima (HTTP / banner).

---

## Testes mais comuns (por objetivo)

### 1) Testar se um serviço/porta está acessível (TCP/UDP)
**Comando base:**
```bash
nc -v IP_OU_HOST PORTA
```

Exemplos:
```bash
# SSH (TCP)
nc -v 10.13.90.158 22

# DNS (UDP)
nc -v -u walk.ns.cloudflare.com 53
```

Para testes rápidos (sem enviar payload) e com timeout curto:
```bash
nc -vz -w2 HOST PORTA
```

> `-v` verbose • `-z` scan (não envia payload) • `-w2` timeout (2s)

---

### 2) Simular servidor (listener) para validar chegada na porta
**TCP listener local:**
```bash
nc -v -l 22
```

De outro host:
```bash
nc HOST 22
```

Use isso para validar:
- regra de firewall liberando porta
- NAT/port-forward
- conectividade entre subnets/VLANs

> Isso **não** emula SSH; apenas confirma que a conexão está chegando na porta correta.

---

### 3) Validar rapidamente um endpoint HTTP (sem curl)
Útil para separar “porta abre” vs “app responde”:
```bash
printf "GET /health HTTP/1.1\r\nHost: HOST\r\n\r\n" | nc -w3 HOST 80
```

---

### 4) Capturar/atender requisições de browser (HTTP simples)
Útil para verificar se o serviço está atrás de firewall/NAT e será acessado via navegador.
```bash
echo 'HTTP/1.1 200 OK\n\n%s' "$(cat index.html)" | nc -v -l -p 8999
```

---

### 5) Confirmar perda de pacotes UDP (voz/vídeo)
> Reuniões (Meet/Zoom/Teams) sofrem com **perda/jitter UDP**. O `nc` ajuda a **reproduzir e medir perda** entre dois pontos.

**Receptor (destino):**
```bash
nc -u -l 9999 > /tmp/udp_rx.txt
```

**Emissor (origem):**
```bash
for i in $(seq 1 2000); do echo "$i"; done | nc -u -w1 DESTINO 9999
```

**Checar “buracos” (perda):**
```bash
sort -n /tmp/udp_rx.txt | awk 'NR==1{p=$1;next} {if($1!=p+1) print "gap",p,"->",$1; p=$1}'
```

Interpretação:
- Se aparecerem vários `gap`, há **perda** no caminho (Wi‑Fi ruim, congestionamento, QoS errado, firewall com rate-limit/inspection etc.).
- Para isolar “onde começa”, repita o teste em segmentos diferentes (por dentro/fora do firewall).

---

### 6) “Banner grabbing” (identificar o que está na porta)
```bash
nc -v HOST 25
nc -v HOST 6379
```
Útil para serviços que respondem com banner/texto (ex.: SMTP, Redis etc.).

---

### 7) Teste de DNS (somente conectividade da porta 53)
```bash
nc -vz -w2 DNS_SERVER 53
```
Se a porta abre mas DNS segue lento: use `dig`, `resolvectl` e/ou `tcpdump`.

---

## Resumo de comandos (cheatsheet)

### Enviar um GET simples (teste rápido de conectividade/handshake)
```bash
echo "GET / HTTP/1.1\r\n\n\n" | nc -v kernel.org 443
```
**Resumo:** abre conexão e envia um payload mínimo; útil para confirmar conectividade. (Para TLS de verdade: `openssl s_client -connect host:443`.)

### Listar portas abertas em um host (range)
```bash
nc -n -z -v 192.168.1.100 20-445
```
**Resumo:** varre um intervalo de portas TCP e mostra as que respondem.

### Listar portas abertas filtrando somente sucesso
```bash
nc -n -z -v 192.168.1.100 20-445 2>&1 | grep succeeded
```
**Resumo:** mesma varredura, filtrando somente resultados de sucesso.

### Transferir diretório via rede usando compressão (tar + nc)
```bash
sudo tar cvzp /var/log/ | nc -v 10.13.90.158 10200
nc -v -l -p 10200 | tar xvzp
```
**Resumo:** envia um stream `tar.gz` por rede e extrai no destino (bom em laboratório/contingência; atenção a políticas de segurança).

---

## Quando netcat NÃO é suficiente (e o que usar)
- **Jitter/perda detalhada, QUIC/RTP, evidência de firewall** → `tcpdump`/Wireshark
- **Rotas** → `traceroute`/`mtr`
- **Wi‑Fi** → métricas do AP, RSSI/SNR, canal/interferência (e `ping`/`mtr`)
- **Bufferbloat/congestionamento** → testes de carga + QoS/Smart Queue
