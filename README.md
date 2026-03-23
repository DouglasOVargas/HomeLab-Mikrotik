# 🏠 HomeLab — MikroTik RB750r2

Documentação completa da configuração do roteador MikroTik do meu HomeLab pessoal, incluindo rede local, WireGuard VPN, Kubernetes, Proxmox e boas práticas de firewall.

---

## 📋 Sumário

- [Hardware](#hardware)
- [Topologia da Rede](#topologia-da-rede)
- [Interfaces](#interfaces)
- [Rede Local (LAN)](#rede-local-lan)
- [WireGuard VPN](#wireguard-vpn)
- [Firewall](#firewall)
- [NAT e Port Forwards](#nat-e-port-forwards)
- [DNS](#dns)
- [Kubernetes](#kubernetes)
- [Serviços Desabilitados](#serviços-desabilitados)
- [Boas Práticas Aplicadas](#boas-práticas-aplicadas)

---

## Hardware

| Item | Detalhe |
|---|---|
| Modelo | MikroTik RB750r2 (hEX) |
| RouterOS | 7.20.4 |


---

## Topologia da Rede

```
Internet (177.74.142.238)
        │
        ▼
  Roteador Upstream
  192.168.100.0/24
        │
        ▼
  MikroTik RB750r2
  wan5 → 192.168.100.220
        │
        ├─── LAN (bridge1_RedeLocal)
        │    10.10.10.0/24
        │    ├── lan01 (ether1)
        │    ├── lan02 (ether2)
        │    ├── lan03 (ether3)
        │    └── lan04 (ether4)
        │
        └─── WireGuard VPN (wg0)
             192.168.200.0/24
```

---

## Interfaces

| Interface | Nome Original | Função |
|---|---|---|
| ether1 | lan01 | LAN — bridge |
| ether2 | lan02 | LAN — bridge |
| ether3 | lan03 | LAN — bridge |
| ether4 | lan04 | LAN — bridge |
| ether5 | wan5 | WAN — DHCP client |
| wg0 | wg0 | WireGuard VPN |

---

## Rede Local (LAN)

- **Subnet:** `10.10.10.0/24`
- **Gateway:** `10.10.10.1`
- **DHCP Pool:** `10.10.10.2 – 10.10.10.199`
- **Lease time:** 8 horas
- **DNS:** `10.10.10.1` (local) + `8.8.8.8` (fallback)

### Leases Fixos (DHCP Static)

| IP | MAC | Descrição |
|---|---|---|
| 10.10.10.188 | BC:24:11:96:B1:06 | Host |
| 10.10.10.189 | BC:24:11:49:67:C8 | Host |
| 10.10.10.193 | BC:24:11:DA:7F:7D | Host (HTTP :8080) |
| 10.10.10.194 | BC:24:11:5C:F3:D7 | Kubernetes Node (gateway K8s) |
| 10.10.10.197 | — | Host (SSH externo :22197) |
| 10.10.10.198 | BC:24:11:1C:48:3E | Host |
| 10.10.10.199 | BC:24:11:E2:56:43 | GitLab (`gitlab.home.lab`) |

---

## WireGuard VPN

Acesso remoto seguro à rede do HomeLab via WireGuard, protocolo moderno e performático que substitui o L2TP/IPsec.

### Configuração do Servidor (MikroTik)

| Parâmetro | Valor |
|---|---|
| Interface | `wg0` |
| Porta | `51820/UDP` |
| Subnet VPN | `192.168.200.0/24` |
| IP do router | `192.168.200.1` |
| MTU | 1420 |
| DDNS | `<SEU_DDNS>.sn.mynetname.net` |

### Peer: douglas-notebook

| Parâmetro | Valor |
|---|---|
| IP do cliente | `192.168.200.2/32` |
| Allowed IPs | `10.10.10.0/24, 192.168.200.0/24` |
| PersistentKeepalive | 25s |

### Configuração do Cliente (Windows)

```ini
[Interface]
PrivateKey = <CHAVE_PRIVADA_DO_CLIENTE>
Address = 192.168.200.2/24
DNS = 10.10.10.1

[Peer]
PublicKey = <PUBLIC_KEY_DO_MIKROTIK>
Endpoint = <SEU_DDNS>.sn.mynetname.net:51820
AllowedIPs = 10.10.10.0/24, 192.168.200.0/24
PersistentKeepalive = 25
```

> ⚠️ **NAT Duplo:** O roteador está atrás de um NAT upstream. Para acesso externo, é necessário configurar port forward `UDP 51820` no roteador upstream apontando para `192.168.100.220`. Para uso interno, use `Endpoint = 192.168.100.220:51820`.

---

## Firewall

### Filter Rules

#### Chain: Forward

| # | Ação | Origem | Destino | Descrição |
|---|---|---|---|---|
| 0 | drop | — | — | Drop pacotes inválidos |
| 1 | accept | 10.10.10.0/24 | 10.0.0.0/16 | LAN → Kubernetes |
| 2 | accept | 10.0.0.0/16 | 10.10.10.0/24 | Kubernetes → LAN |
| 3 | accept | 192.168.200.0/24 | 10.10.10.0/24 | WireGuard → LAN |
| 4 | accept | 10.10.10.0/24 | 192.168.200.0/24 | LAN → WireGuard |
| 5 | drop | — | — | Drop todo o resto |

#### Chain: Input

| # | Ação | Protocolo | Porta | Interface | Descrição |
|---|---|---|---|---|---|
| 0 | accept | UDP | 51820 | wan5 | WireGuard |
| 1 | accept | — | — | — | Established/Related |
| 2 | accept | — | — | — | LAN input (10.10.10.0/24) |
| 3 | accept | — | — | — | WireGuard VPN input |
| 4 | drop | UDP | 53 | wan5 | Bloqueia DNS da WAN |
| 5 | drop | TCP | 53 | wan5 | Bloqueia DNS da WAN |
| 6 | drop | — | — | — | Drop todo o resto |

---

## NAT e Port Forwards

### Masquerade

| Regra | Descrição |
|---|---|
| srcnat → wan5 | NAT geral da LAN para internet |
| srcnat → wan5 (192.168.200.0/24) | NAT dos peers WireGuard |
| srcnat accept LAN → K8s | Sem NAT para tráfego LAN ↔ Kubernetes |

### Port Forwards (DNAT)

| Porta Externa | Protocolo | Destino Interno | Porta Interna | Descrição |
|---|---|---|---|---|
| 8006 | TCP | 10.10.10.190 | 8006 | Proxmox Node 1 |
| 8007 | TCP | 10.10.10.251 | 8006 | Proxmox Node 2 |
| 8008 | TCP | 10.10.10.252 | 8006 | Proxmox Node 3 |
| 22197 | TCP | 10.10.10.197 | 22 | SSH externo |
| 8080 | TCP | 10.10.10.193 | 8080 | HTTP app |

---

## DNS

- **Servidor DNS local** habilitado (`allow-remote-requests=yes`)
- **Upstream DNS:** `8.8.8.8` (Google) + `1.1.1.1` (Cloudflare)
- **DNS bloqueado da WAN** via firewall (evita DNS aberto)

### Registros Estáticos

| Nome | IP | Tipo |
|---|---|---|
| `gitlab.home.lab` | 10.10.10.199 | A |

---

## Kubernetes

- **Pod CIDR:** `10.0.0.0/16`
- **Gateway K8s:** `10.10.10.194` (node principal)
- Rota estática configurada para encaminhar tráfego K8s diretamente ao node

```
/ip route add dst-address=10.0.0.0/16 gateway=10.10.10.194 comment="Kubernetes Pod CIDR"
```

---

## Serviços Desabilitados

| Serviço | Status |
|---|---|
| FTP | ❌ Desabilitado |
| SSH | ❌ Desabilitado |
| Telnet | ❌ Desabilitado |
| WWW (HTTP) | ❌ Desabilitado |
| Winbox | ✅ Ativo na porta `8443` (não padrão) |
| IPv6 | ❌ Desabilitado |
| L2TP Server | ❌ Desabilitado |

---

## Boas Práticas Aplicadas

- ✅ **WireGuard** substituindo L2TP/IPsec — mais leve, seguro e moderno
- ✅ **Drop de pacotes inválidos** no início da chain forward
- ✅ **Drop final** em todas as chains (input e forward)
- ✅ **DNS bloqueado na WAN** — evita que o roteador seja usado como DNS aberto
- ✅ **Porta do Winbox alterada** para `8443` — evita scanners automáticos
- ✅ **SSH, Telnet, FTP e HTTP** desabilitados nos serviços do router
- ✅ **IPv6 desabilitado** — reduz superfície de ataque
- ✅ **DDNS via IP Cloud** — acesso remoto mesmo com IP dinâmico
- ✅ **Leases DHCP fixos** para todos os hosts críticos
- ✅ **Regra No-NAT** para tráfego LAN ↔ Kubernetes (preserva IPs reais)
- ✅ **Connection tracking UDP timeout** reduzido para 10s — melhor performance

---

## 📁 Estrutura do Repositório

```
.
├── README.md
└── backup-config.rsc     # Export completo do RouterOS
```

---

> 📅 Última atualização: Março/2026  
> 🛠️ RouterOS 7.20.4 — MikroTik RB750r2