# 🔐 Certificado Wildcard SSL — Proxmox HomeLab

Guia para aplicar o certificado wildcard `*.home.lab` emitido pela **HomeLab Root CA** (MikroTik) nos nodes Proxmox.

---

## 📁 Certificados gerados pelo MikroTik

| Arquivo | Descrição |
|---|---|
| `cert_export_HomeLab-CA.crt` | Certificado raiz da CA — instalar no Windows/browsers/sistemas |
| `cert_export_wildcard.home.lab.crt` | Certificado público wildcard `*.home.lab` |
| `cert_export_wildcard.home.lab.key` | Chave privada **criptografada** — precisa descriptografar antes de usar |

> ⚠️ **Nunca suba o arquivo `.key` para o GitHub!**

---

## 🖥️ Nodes Proxmox

| Node | Hostname | IP |
|---|---|---|
| proxmox0 | proxmox0.home.lab | 10.10.10.192 |
| proxmox1 | proxmox1.home.lab | 10.10.10.251 |
| proxmox2 | proxmox2.home.lab | 10.10.10.252 |

---

## 🔑 Pré-requisito — Descriptografar a chave privada

O MikroTik exporta a chave criptografada. O Proxmox exige a chave sem criptografia.

Execute **uma vez** no Linux/WSL antes de copiar para os nodes:

```bash
openssl pkey \
  -in cert_export_wildcard.home.lab.key \
  -out wildcard.home.lab.decrypted.key

# Verificar (deve mostrar: -----BEGIN PRIVATE KEY-----)
head -1 wildcard.home.lab.decrypted.key
```

---

## 🚀 Aplicar certificado em cada node

Repita os passos abaixo para cada node substituindo o IP.

### Passo 1 — Copiar arquivos para o node (do Windows/Linux)

```bash
scp cert_export_wildcard.home.lab.crt  root@<IP_DO_NODE>:/tmp/
scp cert_export_HomeLab-CA.crt         root@<IP_DO_NODE>:/tmp/
scp wildcard.home.lab.decrypted.key    root@<IP_DO_NODE>:/tmp/
```

### Passo 2 — Acessar o node via SSH

```bash
ssh root@<IP_DO_NODE>
```

### Passo 3 — Fazer backup dos certificados atuais

```bash
cp /etc/pve/local/pve-ssl.pem /etc/pve/local/pve-ssl.pem.bak
cp /etc/pve/local/pve-ssl.key /etc/pve/local/pve-ssl.key.bak
```

### Passo 4 — Instalar o certificado wildcard

```bash
# Montar bundle (certificado + CA)
cat /tmp/cert_export_wildcard.home.lab.crt \
    /tmp/cert_export_HomeLab-CA.crt \
    > /etc/pve/local/pve-ssl.pem

# Instalar chave privada
cp /tmp/wildcard.home.lab.decrypted.key /etc/pve/local/pve-ssl.key

# Ajustar permissões
chmod 640 /etc/pve/local/pve-ssl.pem
chmod 600 /etc/pve/local/pve-ssl.key
```

### Passo 5 — Validar ANTES de reiniciar

```bash
# Verificar certificado
openssl x509 -in /etc/pve/local/pve-ssl.pem -noout -subject -dates

# Verificar se chave e certificado batem (os dois md5sum devem ser IGUAIS)
openssl x509 -noout -modulus -in /etc/pve/local/pve-ssl.pem | md5sum
openssl rsa  -noout -modulus -in /etc/pve/local/pve-ssl.key | md5sum
```

Resultado esperado:
```
subject=C=BR, O=HomeLab, CN=*.home.lab
notBefore=Mar 26 02:24:15 2026 GMT
notAfter=Jun 28 02:24:15 2028 GMT
64993ace029ed915052d80410c35efaa  -   ← devem ser iguais
64993ace029ed915052d80410c35efaa  -   ← devem ser iguais
```

> ⛔ **Não prossiga se os hashes forem diferentes!** Restaure o backup e revise os arquivos.

### Passo 6 — Reiniciar o proxy do Proxmox

```bash
systemctl restart pveproxy
systemctl status pveproxy
```

### Passo 7 — Instalar a CA no sistema do node

Para que o próprio node confie em outros serviços `.home.lab`:

```bash
cp /tmp/cert_export_HomeLab-CA.crt \
   /usr/local/share/ca-certificates/HomeLab-CA.crt

update-ca-certificates
```

### Passo 8 — Limpeza

```bash
rm /tmp/cert_export_* /tmp/wildcard.*
```

---

## ✅ Validação final

Após aplicar em cada node, acesse via browser:

| Node | URL |
|---|---|
| proxmox0 | https://proxmox0.home.lab:8006 |
| proxmox1 | https://proxmox1.home.lab:8006 |
| proxmox2 | https://proxmox2.home.lab:8006 |

O certificado deve mostrar:
- **Emitido para:** `*.home.lab`
- **Emitido por:** `HomeLab Root CA`

---

## 🪟 Instalar a CA Root no Windows (uma vez só)

Para que o browser confie nos certificados sem aviso:

1. Duplo clique em `cert_export_HomeLab-CA.crt`
2. **Instalar Certificado → Computador Local**
3. **Colocar todos os certificados no repositório a seguir**
4. Selecionar: **Autoridades de Certificação Raiz Confiáveis**
5. Avançar → Concluir
6. Fechar e reabrir o browser (ou rodar `chrome://restart`)

---

## 🔁 Recuperação de emergência

Se o Proxmox ficar inacessível após a troca de certificado:

```bash
# Restaurar backup via SSH ou console
cp /etc/pve/local/pve-ssl.pem.bak /etc/pve/local/pve-ssl.pem
cp /etc/pve/local/pve-ssl.key.bak /etc/pve/local/pve-ssl.key
systemctl restart pveproxy

# Ou resetar para o certificado auto-assinado original
pvenode cert reset --force
systemctl restart pveproxy
```

---

## 📋 DNS no MikroTik

Adicione os registros para acessar os nodes por nome:

```
/ip dns static
add address=10.10.10.192 name=proxmox0.home.lab type=A
add address=10.10.10.251 name=proxmox1.home.lab type=A
add address=10.10.10.252 name=proxmox2.home.lab type=A
```

---

> 📅 Última atualização: Março/2026  
> 🔐 CA gerada pelo MikroTik RB750r2 — RouterOS 7.20.4  
> ✅ Testado no Proxmox VE 8.x
