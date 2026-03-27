# 🌐 Como funciona uma conexão HTTPS — do clique à resposta

> **Wrap-up de Redes de Computadores**  
> Um guia técnico completo sobre o que acontece nos bastidores quando você acessa qualquer site HTTPS.

---

## 📋 Índice

1. [Visão geral do fluxo](#-visão-geral-do-fluxo)
2. [Browser monta o request](#-1-browser-monta-o-request)
3. [DNS — traduzindo nomes em IPs](#-2-dns--traduzindo-nomes-em-ips)
4. [Da aplicação à placa de rede](#-3-da-aplicação-à-placa-de-rede)
5. [Da placa de rede ao roteador — WiFi](#-4-da-placa-de-rede-ao-roteador--wifi)
6. [DHCP — como o dispositivo obteve seu IP](#-5-dhcp--como-o-dispositivo-obteve-seu-ip)
7. [NAT — saindo para a internet](#-6-nat--saindo-para-a-internet)
8. [BGP — roteamento global](#-7-bgp--roteamento-global)
9. [TCP — o canal confiável](#-8-tcp--o-canal-confiável)
10. [TLS — o envelope criptografado](#-9-tls--o-envelope-criptografado)
11. [A resposta chega — de bits a letras](#-10-a-resposta-chega--de-bits-a-letras)
12. [Cookies de sessão](#-11-cookies-de-sessão)
13. [Glossário rápido](#-glossário-rápido)

---

## 🗺 Visão geral do fluxo

```
Você digita https://exemplo.com e aperta Enter

[BROWSER]
  └─ Monta HTTP Request (método + headers + body)
  └─ Resolve DNS (nome → IP)
  └─ Passa para o OS (socket API)

[OS / KERNEL]
  └─ TCP: cria segmentos
  └─ IP: empacota com endereços
  └─ Ethernet/WiFi: adiciona o MAC

[HARDWARE — NIC]
  └─ Converte bits em sinal de rádio (WiFi 2.4/5 GHz)

[ROTEADOR / MODEM]
  └─ DHCP (já fez antes — você tem IP local)
  └─ NAT: troca seu IP privado pelo IP público
  └─ Envia para a internet

[INTERNET]
  └─ BGP: roteadores decidem o melhor caminho
  └─ Pacotes viajam por múltiplos hops

[SERVIDOR]
  └─ TCP aceita conexão
  └─ TLS: handshake, troca de chaves, criptografia
  └─ HTTP processa o request
  └─ Envia response

[VOLTA — resposta chega]
  └─ Sinal de rádio → NIC converte para bits
  └─ TLS descriptografa
  └─ TCP remonta os segmentos
  └─ HTTP/browser interpreta
  └─ Bytes → Unicode → caracteres → pixels na tela
```

---

## 🖥 1. Browser monta o request

Quando você aperta Enter, o browser faz muito trabalho antes de qualquer coisa sair pela rede.

### 1.1 Parsing da URL

```
https://www.exemplo.com:443/produtos?id=42#descricao
│        │               │   │          │   └─ fragment (só browser, não vai ao servidor)
│        │               │   │          └─ query string
│        │               │   └─ path
│        │               └─ porta (443 é o padrão HTTPS)
│        └─ host
└─ scheme (protocolo)
```

O browser identifica que o scheme é `https`, então **já sabe que vai precisar de TLS** antes de qualquer coisa.

### 1.2 HTTP Request

Um request HTTP tem três partes: **linha de request**, **headers** e **body**.

```
GET /produtos?id=42 HTTP/1.1
Host: www.exemplo.com
User-Agent: Mozilla/5.0 (Windows NT 10.0) Chrome/120.0
Accept: text/html,application/xhtml+xml,*/*
Accept-Language: pt-BR,pt;q=0.9,en;q=0.8
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Cookie: session_id=abc123; preferencia=dark_mode
```

**Linha de request:**
- `GET` — método HTTP (GET, POST, PUT, DELETE, PATCH…)
- `/produtos?id=42` — o caminho no servidor
- `HTTP/1.1` — versão do protocolo

**Headers importantes:**
| Header | O que faz |
|---|---|
| `Host` | Qual domínio você quer (necessário em virtual hosting) |
| `User-Agent` | Identifica o browser/SO |
| `Accept` | Quais formatos o browser aceita |
| `Accept-Encoding` | Compressões que suporta (gzip, brotli…) |
| `Cookie` | Envia os cookies salvos para esse domínio |
| `Authorization` | Credenciais (para APIs protegidas) |
| `Content-Type` | Quando há body, diz o formato dos dados |

### 1.3 Métodos HTTP e seus significados semânticos

```
GET     → busca um recurso. Idempotente. Sem body.
POST    → cria ou envia dados. Tem body.
PUT     → substitui um recurso inteiro.
PATCH   → atualiza parte de um recurso.
DELETE  → remove um recurso.
HEAD    → igual ao GET, mas só retorna os headers (sem body).
OPTIONS → pergunta quais métodos o servidor aceita (usado no CORS).
```

### 1.4 O request vai para o kernel

O browser chama a **socket API** do sistema operacional:
```c
socket()   // cria o socket TCP
connect()  // inicia o 3-way handshake TCP com o servidor
send()     // envia os dados do request
recv()     // recebe a resposta
```

O browser **não** lida diretamente com TCP. Ele pede ao OS e o kernel cuida do resto.

---

## 🔍 2. DNS — traduzindo nomes em IPs

O computador precisa saber o IP de `www.exemplo.com` antes de abrir qualquer conexão.

### Ordem de resolução

```
1. Cache local do OS    → /etc/hosts ou cache do sistema
2. Cache do browser     → navegadores mantêm cache próprio
3. Resolver local       → geralmente o roteador (192.168.1.1)
4. DNS Recursivo do ISP → seu provedor tem servidores DNS
5. Root Nameservers     → 13 clusters no mundo (a., b., … m.root-servers.net)
6. TLD Nameserver       → responsável por .com, .br, etc.
7. Authoritative NS     → o servidor DNS do próprio domínio
```

### Exemplo de consulta completa

```
Browser → "quem é www.exemplo.com?"
  → Cache local: não tem
  → DNS Recursivo do ISP:
      → Root NS: "não sei, mas .com é responsabilidade do Verisign"
      → TLD NS (.com): "não sei, mas o NS de exemplo.com é ns1.registro.br"
      → Authoritative NS: "www.exemplo.com = 93.184.216.34"
  ← resposta: 93.184.216.34 (TTL: 3600s)
Browser salva no cache por 3600 segundos
```

**Tipo de registros DNS:**
| Tipo | Função |
|---|---|
| A | IPv4 address |
| AAAA | IPv6 address |
| CNAME | Alias para outro nome |
| MX | Mail server |
| TXT | Texto livre (usado para SPF, DKIM, verificações) |
| NS | Aponta para os nameservers do domínio |

---

## 🔌 3. Da aplicação à placa de rede

O caminho dos dados dentro do seu computador segue o modelo de camadas.

### O modelo TCP/IP na prática

```
┌─────────────────────────────┐
│  APLICAÇÃO (HTTP/HTTPS)     │  ← browser, o que você escreve
│  dados: "GET /index HTTP/1.1"│
├─────────────────────────────┤
│  TRANSPORTE (TCP)           │  ← kernel
│  + porta de origem (ex: 52341)│
│  + porta de destino (443)   │
│  + número de sequência      │
│  + checksum                 │
│  = SEGMENTO TCP             │
├─────────────────────────────┤
│  REDE (IP)                  │  ← kernel
│  + IP de origem (192.168.1.5)│
│  + IP de destino (93.184.216.34)│
│  + TTL, protocolo           │
│  = PACOTE IP                │
├─────────────────────────────┤
│  ENLACE (Ethernet/WiFi)     │  ← driver da NIC
│  + MAC de origem            │
│  + MAC de destino (do roteador)│
│  + tipo (0x0800 = IPv4)     │
│  = FRAME                    │
└─────────────────────────────┘
```

Cada camada **encapsula** a camada anterior. É como colocar cartas dentro de envelopes — o envelope maior não sabe o que está escrito na carta interna.

### TCP — garantia de entrega

O TCP resolve três problemas: **confiabilidade**, **ordem** e **controle de fluxo**.

**Three-Way Handshake (abertura da conexão):**
```
Client                    Server
  │                          │
  │──── SYN (seq=100) ──────▶│   "quero conectar"
  │                          │
  │◀─── SYN-ACK (seq=200, ack=101) ──│   "tudo bem, pode vir"
  │                          │
  │──── ACK (ack=201) ──────▶│   "confirmado"
  │                          │
  │══════ conexão aberta ════│
```

**Flags TCP:**
```
SYN   → iniciar conexão
ACK   → confirmar recebimento
FIN   → encerrar conexão
RST   → resetar conexão forçadamente
PSH   → entregue ao processo imediatamente
URG   → dados urgentes
```

**Números de sequência e ACK:**
```
Dados enviados:   [seg 1: bytes 1-1460] [seg 2: bytes 1461-2920]
ACKs recebidos:   ACK=1461 (recebi até 1460, mande a partir de 1461)
                  ACK=2921 (recebi tudo até 2920)
```

Se um segmento se perde, o TCP retransmite. O UDP **não** faz isso — é "fire and forget".

---

## 📡 4. Da placa de rede ao roteador — WiFi

O frame Ethernet precisa chegar fisicamente ao roteador. Se você estiver no WiFi, isso vira um sinal de rádio.

### Como o WiFi funciona

**O sinal de rádio:**
- WiFi 2.4 GHz usa frequências entre 2400 e 2483 MHz
- WiFi 5 GHz usa frequências entre 5150 e 5850 MHz
- WiFi 6 (802.11ax) também usa 6 GHz (6 GHz band)

**Modulação — como bits viram frequência:**

A NIC usa técnicas de modulação para codificar 0s e 1s em ondas de rádio:

```
QAM (Quadrature Amplitude Modulation)
  → combina variação de amplitude + fase da onda
  → WiFi 6 usa 1024-QAM: cada símbolo carrega 10 bits

OFDM (Orthogonal Frequency Division Multiplexing)
  → divide o canal em várias subportadoras (sub-carriers)
  → cada subportadora carrega parte dos dados em paralelo
  → evita interferências entre portadoras (são ortogonais)
```

**Do digital ao analógico (transmissão):**
```
Bits (0101101...) 
  → Codificador: agrupa bits em símbolos
  → Modulador: mapeia símbolos em posições I/Q (amplitude + fase)
  → DAC (Digital-to-Analog Converter): gera sinal elétrico analógico
  → Amplificador: aumenta a potência
  → Antena: converte sinal elétrico em onda eletromagnética
```

**Do analógico ao digital (recepção — roteador recebendo):**
```
Onda eletromagnética chega na antena do roteador
  → ADC (Analog-to-Digital Converter): amostragem do sinal
  → Demodulador: extrai posições I/Q
  → Decodificador: converte símbolos de volta para bits
  → Verificação de CRC: checa se houve erros
  → Frame Ethernet reconstruído
```

### Colisões e CSMA/CA

No WiFi, vários dispositivos compartilham o mesmo meio físico. O protocolo **CSMA/CA** (Carrier Sense Multiple Access with Collision Avoidance) coordena isso:

```
Dispositivo quer transmitir:
  1. Escuta o canal — está livre?
  2. Se ocupado: espera + backoff aleatório
  3. Se livre: espera DIFS (tempo de segurança)
  4. Transmite o frame
  5. Espera ACK do ponto de acesso
  6. Se não receber ACK: assume colisão, aumenta o backoff e tenta de novo
```

---

## 🏠 5. DHCP — como o dispositivo obteve seu IP

Antes de tudo isso acontecer, seu dispositivo precisou de um endereço IP. Isso foi feito pelo **DHCP (Dynamic Host Configuration Protocol)**.

### O processo DORA

```
Cliente (seu PC)              Servidor DHCP (seu roteador)
       │                              │
       │── DISCOVER (broadcast) ─────▶│  "tem alguém aí? preciso de IP"
       │   src: 0.0.0.0               │
       │   dst: 255.255.255.255       │
       │                              │
       │◀─────────── OFFER ──────────│  "tenho o IP 192.168.1.5 para você"
       │                              │  (reservado por 24h)
       │                              │
       │── REQUEST (broadcast) ──────▶│  "quero o 192.168.1.5"
       │                              │
       │◀──────────── ACK ───────────│  "confirmado, é seu por 86400s"
       │                              │
       │ Agora tem:                   │
       │  IP: 192.168.1.5             │
       │  Máscara: 255.255.255.0      │
       │  Gateway: 192.168.1.1        │
       │  DNS: 8.8.8.8                │
```

O lease tem tempo de expiração. Quando está na metade do tempo, o cliente tenta renovar enviando um REQUEST diretamente ao servidor DHCP.

---

## 🔄 6. NAT — saindo para a internet

Seu IP local (`192.168.1.5`) é privado e não pode ser roteado na internet. O roteador usa **NAT (Network Address Translation)** para resolver isso.

### Como o NAT funciona

```
SAÍDA:
  Pacote: src=192.168.1.5:52341 → dst=93.184.216.34:443
  NAT substitui: src=187.10.50.123:61000 → dst=93.184.216.34:443
  Salva na tabela: {187.10.50.123:61000 ↔ 192.168.1.5:52341}

CHEGADA (resposta):
  Pacote: src=93.184.216.34:443 → dst=187.10.50.123:61000
  NAT consulta tabela: 61000 → 192.168.1.5:52341
  NAT substitui: src=93.184.216.34:443 → dst=192.168.1.5:52341
  Encaminha para o dispositivo correto
```

**Por que existe o NAT?**

O espaço de IPs IPv4 (32 bits = ~4,3 bilhões de endereços) não é suficiente para todos os dispositivos do mundo. O NAT permite que milhões de dispositivos compartilhem um único IP público. Com IPv6 (128 bits), a necessidade de NAT tecnicamente desaparece.

---

## 🌍 7. BGP — roteamento global

Seu pacote saiu do roteador com IP de destino `93.184.216.34`. Como ele sabe para onde ir?

### O que é BGP

**BGP (Border Gateway Protocol)** é o protocolo que conecta os sistemas autônomos (AS) da internet entre si. Um AS é uma coleção de redes controladas por uma única organização (ISP, empresa, universidade…).

```
AS 7738 (Telemar/Oi, Brasil)
  └─ conectado a ─▶ AS 3549 (Level3/Lumen, EUA)
                      └─ conectado a ─▶ AS 15133 (Edgecast/Verizon)
                                          └─ 93.184.216.34 (exemplo.com)
```

### Como o BGP decide o caminho

Cada AS anuncia para seus vizinhos os prefixos de rede que ele consegue alcançar:
```
"Eu (AS 7738) consigo alcançar 93.184.0.0/16 passando por AS 3549"
```

Os roteadores BGP escolhem o melhor caminho baseados em atributos como:
- **AS_PATH**: preferir o caminho com menos saltos de AS
- **LOCAL_PREF**: preferência configurada localmente (rota paga vs gratuita)
- **MED (Multi-Exit Discriminator)**: dica do AS vizinho sobre qual entrada usar

Os pacotes não necessariamente seguem o caminho mais curto geograficamente — BGP é sobre **políticas comerciais e técnicas**, não distância física.

---

## 🔒 8. TCP — o canal confiável

O TCP Handshake (seção 3) abre a conexão. Agora os dados fluem em segmentos.

### Controle de congestionamento

O TCP monitora a rede e ajusta a velocidade de envio:
```
Slow Start:
  janela de congestionamento começa pequena (1 MSS)
  duplica a cada RTT enquanto não há perda

Congestion Avoidance:
  após atingir o threshold, cresce mais devagar (linear)

Fast Retransmit:
  se receber 3 ACKs duplicados, retransmite o segmento perdido
  sem esperar o timeout

Fast Recovery:
  reduz a janela pela metade (não vai a zero)
  retoma congestion avoidance
```

**MSS (Maximum Segment Size):** geralmente 1460 bytes em Ethernet (1500 MTU - 20 IP header - 20 TCP header).

---

## 🔐 9. TLS — o envelope criptografado

HTTPS = HTTP + TLS. O TLS garante **confidencialidade**, **integridade** e **autenticação** da conexão.

### TLS 1.3 Handshake

```
Client                                      Server
  │                                            │
  │── ClientHello ────────────────────────────▶│
  │   versão TLS 1.3                           │
  │   cipher suites suportados                 │
  │   extensões (SNI, ALPN...)                 │
  │   key_share (chave pública efêmera DH)     │
  │                                            │
  │◀─────────────────────── ServerHello ───────│
  │                           versão acordada  │
  │                           cipher escolhido │
  │                           key_share server │
  │                                            │
  │  [ambos calculam o segredo compartilhado]  │
  │  usando Diffie-Hellman:                    │
  │  shared_secret = client_priv × server_pub  │
  │  (ou vice-versa — o resultado é igual)     │
  │                                            │
  │◀───────────── {Certificate} ───────────────│  ← criptografado!
  │               {CertificateVerify}          │
  │               {Finished}                  │
  │                                            │
  │── {Finished} ─────────────────────────────▶│
  │                                            │
  │══════════ dados HTTP criptografados ═══════│
```

**SNI (Server Name Indication):** enviado no ClientHello para dizer ao servidor qual certificado usar (um IP pode hospedar vários domínios).

### Certificados e PKI

```
Certificado do servidor contém:
  - Chave pública do servidor
  - Nome do domínio (Common Name / SANs)
  - Validade (not before / not after)
  - Assinatura da CA (Certificate Authority)

Cadeia de confiança:
  exemplo.com cert ──assinado por──▶ Let's Encrypt R3 (CA intermediária)
                                          └─assinado por──▶ ISRG Root X1 (Root CA)
                                                               └─ no trust store do OS/browser
```

O browser verifica a assinatura subindo na cadeia até encontrar uma Root CA que ele já confia.

### Criptografia simétrica (dados em trânsito)

Após o handshake, os dados são protegidos com **AES-GCM** (ou ChaCha20-Poly1305):

```
AES-256-GCM:
  - AES (Advanced Encryption Standard): cifra de bloco de 128 bits
  - 256 bits de chave → 2^256 combinações possíveis
  - GCM (Galois/Counter Mode): adiciona autenticação (AEAD)

Cada registro TLS:
  ┌──────────┬──────────────────────┬─────────────┐
  │ Header   │ Dados criptografados │ Auth Tag    │
  │ (5 bytes)│ (payload)            │ (16 bytes)  │
  └──────────┴──────────────────────┴─────────────┘
  
  O Auth Tag verifica que ninguém alterou os dados no caminho.
  Se o tag não bater: conexão encerrada imediatamente.
```

### Perfect Forward Secrecy (PFS)

O TLS 1.3 usa chaves efêmeras (descartadas após a sessão). Se a chave privada do servidor for comprometida no futuro, **sessões anteriores não podem ser descriptografadas**. Isso se chama Perfect Forward Secrecy.

---

## 📥 10. A resposta chega — de bits a letras

O servidor processou o request e enviou a resposta. Agora vamos ver o caminho de volta até virar texto na tela.

### 10.1 Do sinal de rádio aos bits

```
Antena do seu dispositivo recebe a onda eletromagnética
  │
  ▼
ADC — amostragem: mede a amplitude/fase do sinal em intervalos regulares
  (ex: 802.11ax amostra com resolução de 80 MHz)
  │
  ▼
Demodulação QAM/OFDM — cada símbolo vira um grupo de bits
  (1024-QAM: cada símbolo = 10 bits)
  │
  ▼
Decodificação FEC — Forward Error Correction corrige erros de transmissão
  (WiFi usa códigos como LDPC ou BCC)
  │
  ▼
Verificação CRC (Cyclic Redundancy Check) do frame
  32 bits de checksum no final do frame Ethernet
  se o CRC não bater → frame descartado silenciosamente
  │
  ▼
Frame Ethernet reconstruído — bits organizados em bytes
```

### 10.2 Subindo as camadas

```
Frame Ethernet
  ├─ Remove header/trailer Ethernet (MAC src, MAC dst, tipo, CRC)
  └─ Extrai pacote IP

Pacote IP
  ├─ Verifica checksum do header IP
  ├─ Confere que o IP destino é o seu
  └─ Extrai segmento TCP

Segmento TCP
  ├─ Verifica checksum TCP
  ├─ Confirma número de sequência (está na ordem certa?)
  ├─ Envia ACK de volta para o servidor
  └─ Passa dados para o buffer da aplicação

TLS
  ├─ Verifica o Auth Tag (GCM)
  ├─ Descriptografa com a chave de sessão (AES-256)
  └─ Entrega plaintext HTTP ao browser

HTTP
  ├─ Lê status line: HTTP/1.1 200 OK
  ├─ Processa headers (Content-Type, Content-Encoding...)
  └─ Se Content-Encoding: gzip → descomprime o body
```

### 10.3 Resposta HTTP

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Content-Encoding: gzip
Content-Length: 4823
Cache-Control: max-age=3600
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Lax

[body: bytes comprimidos com gzip]
```

### 10.4 De bytes para Unicode para caracteres

Após a descompressão, o browser tem um array de bytes. O header `Content-Type: text/html; charset=UTF-8` diz como interpretar esses bytes.

**UTF-8 — codificação de caracteres:**

```
ASCII puro (1 byte por caractere — 0x00 a 0x7F):
  'A' → 0x41 → 01000001
  'a' → 0x61 → 01100001
  '0' → 0x30 → 00110000

Caracteres latinos acentuados (2 bytes):
  'ã' → U+00E3 → 0xC3 0xA3 → 11000011 10100011
  'é' → U+00E9 → 0xC3 0xA9 → 11000011 10101001

Emojis e outros (4 bytes):
  '😀' → U+1F600 → 0xF0 0x9F 0x98 0x80

Estrutura UTF-8:
  1 byte:  0xxxxxxx                 (0 a 127 — ASCII compatível)
  2 bytes: 110xxxxx 10xxxxxx        (128 a 2047)
  3 bytes: 1110xxxx 10xxxxxx 10xxxxxx (2048 a 65535)
  4 bytes: 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx (65536 a 1114111)
```

**O Unicode em si:**

Unicode é uma tabela com mais de 140.000 caracteres, cada um com um code point (número). UTF-8 é a forma de *codificar* esses números em bytes. Outras codificações para o mesmo Unicode seriam UTF-16, UTF-32, Latin-1…

```
Bytes recebidos: 0x48 0x65 0x6C 0x6C 0x6F 0x2C 0x20 0x4D 0x75 0x6E 0x64 0x6F
Decodifica UTF-8: H    e    l    l    o    ,       M    u    n    d    o
```

### 10.5 Rendering pipeline (simplificado)

```
HTML bytes
  → Parser HTML → DOM (Document Object Model)
  → Parser CSS → CSSOM
  → DOM + CSSOM → Render Tree
  → Layout: calcula posição/tamanho de cada elemento
  → Paint: decide as cores e estilos
  → Composite: monta as camadas e envia para GPU
  → Pixels na tela
```

---

## 🍪 11. Cookies de sessão

Cookies são a forma como os sites "lembram" quem você é entre requisições. HTTP é um protocolo **stateless** — cada request é independente, o servidor não tem memória nativa. Cookies resolvem isso.

### Como funciona o login com cookies

```
PASSO 1 — Login:
  POST /login HTTP/1.1
  Content-Type: application/json
  
  {"usuario": "joao", "senha": "hunter2"}

PASSO 2 — Servidor cria sessão:
  - Gera session_id aleatório e criptograficamente seguro: "a3f9b2c1d4e5..."
  - Salva no banco: sessions["a3f9b2c1d4e5"] = {user_id: 42, criado: now()}
  - Responde com:
  
  HTTP/1.1 200 OK
  Set-Cookie: session_id=a3f9b2c1d4e5; HttpOnly; Secure; SameSite=Lax; Max-Age=86400; Path=/

PASSO 3 — Browser salva o cookie:
  - Associa o cookie ao domínio (exemplo.com)
  - Guarda no cookie store do browser

PASSO 4 — Próxima requisição (já logado):
  GET /dashboard HTTP/1.1
  Host: exemplo.com
  Cookie: session_id=a3f9b2c1d4e5   ← browser envia automaticamente

PASSO 5 — Servidor verifica:
  - Lê session_id do header Cookie
  - Consulta banco: sessions["a3f9b2c1d4e5"] = {user_id: 42}
  - Encontrou! Sabe que é o usuário 42 (João)
  - Serve o dashboard personalizado
```

### Atributos de segurança dos cookies

| Atributo | O que faz |
|---|---|
| `HttpOnly` | JavaScript não pode ler o cookie (protege contra XSS) |
| `Secure` | Só enviado em conexões HTTPS |
| `SameSite=Lax` | Não enviado em requests cross-site (protege contra CSRF) |
| `SameSite=Strict` | Só enviado em requisições do mesmo site |
| `Max-Age` | Tempo de vida em segundos (após isso, browser apaga) |
| `Domain` | Quais subdomínios recebem o cookie |
| `Path` | Em quais caminhos o cookie é enviado |

### Tipos de cookies

```
Session cookie:
  → Sem Max-Age/Expires
  → Apagado quando o browser fecha
  → Usado para login temporário

Persistent cookie:
  → Tem Max-Age ou Expires
  → Sobrevive ao fechamento do browser
  → Usado para "lembrar de mim"

Third-party cookie:
  → Domínio diferente da página atual
  → Usado por redes de ads e analytics
  → Sendo bloqueado progressivamente pelos browsers

HttpOnly cookie:
  → Não acessível via document.cookie no JS
  → Proteção essencial contra XSS para cookies de sessão
```

### Por que o session_id precisa ser imprevisível?

Se o `session_id` fosse um número sequencial (`session_id=42`), um atacante poderia tentar `session_id=41`, `session_id=43` e sequestrar sessões de outros usuários. Por isso os IDs devem ser:
- Gerados com **CSPRNG** (Cryptographically Secure Pseudo-Random Number Generator)
- Ter entropia suficiente (mínimo 128 bits)
- Ser descartados após logout

---

## 📖 Glossário rápido

| Termo | Definição |
|---|---|
| **ACK** | Acknowledgement — confirmação de recebimento no TCP |
| **AES** | Advanced Encryption Standard — cifra simétrica usada no TLS |
| **ADC** | Analog-to-Digital Converter — converte sinal analógico em bits |
| **AS** | Autonomous System — rede sob controle de uma organização |
| **BGP** | Border Gateway Protocol — roteamento entre ASes |
| **CA** | Certificate Authority — entidade que emite certificados TLS |
| **CSMA/CA** | Protocolo de acesso ao meio no WiFi |
| **DHCP** | Dynamic Host Configuration Protocol — atribuição automática de IPs |
| **DNS** | Domain Name System — resolve nomes em IPs |
| **FEC** | Forward Error Correction — correção de erros sem retransmissão |
| **GCM** | Galois/Counter Mode — modo autenticado do AES |
| **HTTP** | Hypertext Transfer Protocol — protocolo de aplicação |
| **MTU** | Maximum Transmission Unit — tamanho máximo de um frame (1500 bytes em Ethernet) |
| **MSS** | Maximum Segment Size — tamanho máximo de dados em um segmento TCP |
| **NAT** | Network Address Translation — troca IPs privados por públicos |
| **NIC** | Network Interface Card — placa de rede |
| **OFDM** | Modulação por múltiplas subportadoras ortogonais |
| **PKI** | Public Key Infrastructure — sistema de certificados e CAs |
| **QAM** | Quadrature Amplitude Modulation — técnica de modulação |
| **RTT** | Round-Trip Time — tempo ida e volta na rede |
| **SNI** | Server Name Indication — extensão TLS para virtual hosting |
| **TCP** | Transmission Control Protocol — transporte confiável |
| **TLS** | Transport Layer Security — criptografia sobre TCP |
| **TTL** | Time to Live — contador de hops em pacotes IP |
| **UDP** | User Datagram Protocol — transporte sem garantia de entrega |
| **Unicode** | Padrão de codificação de todos os caracteres do mundo |
| **UTF-8** | Codificação de Unicode em bytes (compatível com ASCII) |
| **XSS** | Cross-Site Scripting — injeção de scripts maliciosos |
| **CSRF** | Cross-Site Request Forgery — ataque que forja requisições |

---

## 📺 Vídeo complementar

Este repositório tem um vídeo no YouTube explicando todo esse fluxo em inglês, com diagramas visuais:

> 🔗 **[How HTTPS Works — From Keystroke to Encrypted Bits and Back](https://www.youtube.com/)**  
> *(adicionar link aqui após publicar)*

---

*Feito com 💙 como parte do wrap-up de Redes de Computadores.*
