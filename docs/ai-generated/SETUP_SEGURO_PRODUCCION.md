# 🤖 OpenClaw Setup Seguro: Asistente de Código + GitHub + Crypto Wallet

> **Guía Completa para Producción**  
> Setup: VPS Ubuntu + Docker + GitHub Integration + Wallet Cripto  
> Caso de Uso: Asistente IA que crea código, hace PRs, accede a servicios externos, recibe pagos  
> Fecha: 1 de Febrero, 2026

---

> ⚠️ **NOTA IMPORTANTE**: Esta guía usa "Moltbook" como ejemplo placeholder para cualquier
> servicio externo de chat/comunicación que quieras integrar (puede ser tu propio servicio,
> Discord, Slack, o cualquier API de chat). Adapta las URLs y configuraciones según tu
> servicio real.

---

## 📋 Índice

1. [Arquitectura Completa](#-arquitectura-completa)
2. [Requisitos Previos](#-requisitos-previos)
3. [Setup Paso a Paso](#-setup-paso-a-paso)
4. [Configuración de Secrets](#-configuración-de-secrets)
5. [GitHub Integration](#-github-integration)
6. [Moltbook Integration](#-moltbook-integration)
7. [Crypto Wallet](#-crypto-wallet)
8. [Auditoría de PRs](#-auditoría-de-prs)
9. [Security Hardening](#-security-hardening)
10. [Monitoreo](#-monitoreo)
11. [Troubleshooting](#-troubleshooting)

---

## 🏗️ Arquitectura Completa

### Diagrama General

```
Tu PC (Windows 11)
└── [Sin acceso directo]

VPS Ubuntu (Producción)
└── Docker Network: openclaw-secure
    ├── Container: openclaw-agent
    │   ├── GitHub Client (Token)
    │   ├── Moltbook Client (API Key)
    │   ├── Wallet Client (Private Key encriptada)
    │   └── Code Sandbox (Vitest)
    │
    ├── Secrets Manager
    │   ├── GitHub Token
    │   ├── Moltbook API Key
    │   ├── Wallet Private Key (encriptado)
    │   └── Gemini API Key
    │
    └── Monitoring
        ├── Logs (ELK o similar)
        ├── Billing Alerts
        ├── PR Audit Log
        └── Wallet Activity Log
```

---

## ✅ Requisitos Previos

### Antes de Empezar, Necesitas:

```
☐ VPS Ubuntu 22.04 LTS (mínimo 2GB RAM, 20GB SSD)
☐ Docker & Docker Compose instalados
☐ GitHub Account + Personal Access Token
☐ Gemini API Key (Google Cloud)
☐ Moltbook Account + API Key
☐ Wallet Cripto (Solana dev account recomendado para tests)
☐ Domain name opcional (para webhooks)
☐ SSH key para VPS (no contraseña)
```

### Costo Estimado (Mensual):

```
VPS (Linode/DO):        $5-10/mes
Gemini API (free tier): $0 (primero)
GitHub:                 $0 (público)
Wallet:                 $0 (solo fees de transacción)
─────────────────────────────────
Total:                  $5-10/mes
```

---

## 🚀 Setup Paso a Paso

### FASE 1: VPS Inicial & Security Hardening

```bash
# 1. SSH a tu VPS
ssh root@your_vps_ip

# 2. Updates
apt update && apt upgrade -y

# 3. Crear usuario no-root
adduser openclaw
usermod -aG sudo openclaw
su - openclaw

# 4. SSH key setup (en tu PC)
ssh-keygen -t ed25519 -f ~/.ssh/openclaw_vps -C "openclaw@vps"

# 5. Agregar public key al VPS
cat ~/.ssh/openclaw_vps.pub | ssh-copy-id -i - openclaw@your_vps_ip

# 6. Configurar SSH (como root)
sudo nano /etc/ssh/sshd_config
# Cambiar:
# PermitRootLogin no
# PasswordAuthentication no
# PubkeyAuthentication yes

sudo systemctl restart ssh

# 7. Firewall
sudo ufw enable
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp         # SSH
sudo ufw allow 80/tcp         # HTTP (webhooks)
sudo ufw allow 443/tcp        # HTTPS (webhooks)

# 8. Fail2ban
sudo apt install fail2ban -y
sudo systemctl enable fail2ban

# 9. Actualizaciones automáticas
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades

# 10. Verificar
sudo ufw status
sudo systemctl status fail2ban
```

### FASE 2: Docker Setup

```bash
# 1. Instalar Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 2. Agregar usuario al grupo docker
sudo usermod -aG docker openclaw

# 3. Instalar Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 4. Crear directorios
mkdir -p ~/openclaw-setup/{secrets,config,data,logs}
cd ~/openclaw-setup

# 5. Crear estructura
mkdir -p secrets config data/sessions data/logs
touch .env.example
```

### FASE 3: Estructura de Archivos

```bash
~/openclaw-setup/
├── docker-compose.yml          # Principal
├── .env.example                # Plantilla (sin secrets)
├── Dockerfile                  # Custom image
├── secrets/
│   ├── github_token.txt        # Creado después
│   ├── gemini_key.txt
│   ├── moltbook_api_key.txt
│   ├── wallet_private_key.enc  # Encriptado
│   └── .gitignore              # IMPORTANTE
├── config/
│   ├── openclaw.yaml           # Config principal
│   └── tools-policy.yaml       # Policy de tools
├── data/
│   ├── sessions/               # Sesiones guardadas
│   └── logs/                   # Logs persistentes
└── scripts/
    ├── setup-secrets.sh        # Helper para crear secrets
    ├── test-github.sh
    ├── test-wallet.sh
    └── backup.sh
```

---

## 🔐 Configuración de Secrets

### Paso 1: Crear Archivo .env.example

```bash
# ~/openclaw-setup/.env.example

# NOTA: Este archivo NO tiene valores reales
# Copiar a .env local y NUNCA commitar

# Gemini API
GEMINI_API_KEY=tu-api-key-aqui
GEMINI_MODEL=gemini-2.0-flash

# GitHub
GITHUB_TOKEN=ghp_xxxxxxxxxxxxx
GITHUB_USERNAME=tu-usuario
GITHUB_REPO_WHITELIST=repo1,repo2,repo3

# Moltbook
MOLTBOOK_API_KEY=mb_xxxxxxxxxxxxx
MOLTBOOK_BASE_URL=https://api.moltbook.com

# Wallet Cripto
WALLET_ADDRESS=Solana-public-address
WALLET_PRIVATE_KEY_ENCRYPTED=encrypted-key-here
WALLET_ENCRYPTION_PASSWORD=strong-password
WALLET_NETWORK=devnet  # o mainnet

# OpenClaw Config
OPENCLAW_PROFILE=production
LOG_LEVEL=info
NODE_ENV=production
```

### Paso 2: Script para Crear Secrets Encriptados

```bash
#!/bin/bash
# ~/openclaw-setup/scripts/setup-secrets.sh

set -e

SECRETS_DIR="./secrets"
echo "🔐 Configurando secrets de forma segura..."

# 1. GitHub Token
echo -e "\n📝 GitHub Token:"
echo "1. Ve a: https://github.com/settings/tokens"
echo "2. Crea Personal Access Token con scopes:"
echo "   - repo (full control)"
echo "   - read:user"
echo "3. Pega el token:"
read -s GITHUB_TOKEN
echo "$GITHUB_TOKEN" > "$SECRETS_DIR/github_token.txt"
chmod 600 "$SECRETS_DIR/github_token.txt"
echo "✅ GitHub token guardado"

# 2. Gemini API Key
echo -e "\n📝 Gemini API Key:"
echo "1. Ve a: https://aistudio.google.com/app/apikeys"
echo "2. Copia tu API Key"
echo "3. Pega aquí:"
read -s GEMINI_KEY
echo "$GEMINI_KEY" > "$SECRETS_DIR/gemini_key.txt"
chmod 600 "$SECRETS_DIR/gemini_key.txt"
echo "✅ Gemini API key guardado"

# 3. Moltbook API Key
echo -e "\n📝 Moltbook API Key:"
echo "1. Ve a: https://moltbook.com/settings/api"
echo "2. Copia tu API Key"
echo "3. Pega aquí:"
read -s MOLTBOOK_KEY
echo "$MOLTBOOK_KEY" > "$SECRETS_DIR/moltbook_api_key.txt"
chmod 600 "$SECRETS_DIR/moltbook_api_key.txt"
echo "✅ Moltbook API key guardado"

# 4. Wallet Private Key (Encriptado)
echo -e "\n📝 Wallet Private Key (será encriptado):"
echo "1. Genera o copia tu private key (base58 para Solana)"
echo "2. Pega aquí:"
read -s WALLET_KEY
echo "3. Ingresa contraseña para encriptar:"
read -s WALLET_PASSWORD

# Encriptar con OpenSSL
echo "$WALLET_KEY" | openssl enc -aes-256-cbc -salt -out "$SECRETS_DIR/wallet_private_key.enc" -k "$WALLET_PASSWORD"
chmod 600 "$SECRETS_DIR/wallet_private_key.enc"
echo "✅ Wallet private key guardado (encriptado)"

# 5. .gitignore
echo -e "\n🔒 Creando .gitignore para secrets..."
cat > "$SECRETS_DIR/.gitignore" << 'EOF'
*.txt
*.enc
*.key
*.pem
.env
.env.local
EOF

echo -e "\n✅ Todos los secrets configurados"
echo "🔒 Archivos guardados en: $SECRETS_DIR/"
echo "⚠️  NUNCA hagas git add en esta carpeta"
```

### Paso 3: Ejecutar Script

```bash
chmod +x scripts/setup-secrets.sh
./scripts/setup-secrets.sh
```

---

## 🐳 Docker Compose Configuration

### docker-compose.yml Completo

```yaml
version: '3.9'

services:
  openclaw:
    build:
      context: .
      dockerfile: Dockerfile
    
    container_name: openclaw-agent
    restart: unless-stopped
    
    # Secrets (NO env vars)
    secrets:
      - github_token
      - gemini_key
      - moltbook_api_key
      - wallet_private_key
    
    # Variables de configuración (sin valores sensibles)
    environment:
      NODE_ENV: production
      LOG_LEVEL: info
      GITHUB_USERNAME: ${GITHUB_USERNAME}
      GITHUB_REPO_WHITELIST: ${GITHUB_REPO_WHITELIST}
      MOLTBOOK_BASE_URL: ${MOLTBOOK_BASE_URL}
      WALLET_ADDRESS: ${WALLET_ADDRESS}
      WALLET_NETWORK: ${WALLET_NETWORK:-devnet}
    
    # Volúmenes
    volumes:
      # Sessions (persistente)
      - ./data/sessions:/home/openclaw/.openclaw/sessions:rw
      
      # Logs (persistente)
      - ./data/logs:/home/openclaw/.openclaw/logs:rw
      
      # Config (read-only)
      - ./config:/home/openclaw/config:ro
      
      # Temp (ephemeral)
      - /tmp:/tmp
    
    # Networking
    networks:
      - openclaw-net
    
    # Resources
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 1G
        reservations:
          cpus: '1'
          memory: 512M
    
    # Security
    user: 1000:1000
    read_only: true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    security_opt:
      - no-new-privileges:true
    
    # Logging
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

networks:
  openclaw-net:
    driver: bridge

secrets:
  github_token:
    file: ./secrets/github_token.txt
  gemini_key:
    file: ./secrets/gemini_key.txt
  moltbook_api_key:
    file: ./secrets/moltbook_api_key.txt
  wallet_private_key:
    file: ./secrets/wallet_private_key.enc
```

### Dockerfile Personalizado

```dockerfile
FROM node:22-alpine

WORKDIR /app

# Crear usuario no-root
RUN addgroup -g 1000 openclaw && \
    adduser -D -u 1000 -G openclaw openclaw

# Instalar OpenClaw
RUN npm install -g openclaw@latest

# Instalar herramientas necesarias
RUN apk add --no-cache \
    git \
    bash \
    curl \
    openssl

# Crear directorios
RUN mkdir -p /home/openclaw/.openclaw/sessions && \
    mkdir -p /home/openclaw/.openclaw/logs && \
    chown -R openclaw:openclaw /home/openclaw

# Cambiar a usuario no-root
USER openclaw

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:18789/health || exit 1

# Entrypoint
CMD ["openclaw", "gateway", "--bind", "0.0.0.0", "--port", "18789", "--verbose"]
```

---

## 🐙 GitHub Integration

### Paso 1: Crear GitHub Token

```bash
# En GitHub:
# 1. Settings → Developer settings → Personal access tokens
# 2. Tokens (classic)
# 3. Generate new token

# Scopes necesarios:
- repo (full control of repositories)
- read:user (read user profile)
- write:repo_hook (para webhooks si quieres)
```

### Paso 2: Configurar en OpenClaw

```yaml
# config/openclaw.yaml

github:
  enabled: true
  api_version: 3
  
  # Token via secret
  token_path: /run/secrets/github_token
  
  # Configuración de seguridad
  repository_whitelist:
    - myusername/my-project
    - myusername/another-project
    # NO: repos que no quieres que toque
  
  pr_settings:
    # Tipo de PRs que puede crear
    auto_create_pr: true
    require_human_review: true
    
    # Límites
    max_pr_per_day: 5
    max_files_per_pr: 10
    
    # Branches permitidas
    target_branches:
      - develop
      - feature/*
    # NO: main, production
    
    # Descripción obligatoria
    require_description: true
    require_tests: true
    
    labels:
      - "ai-generated"
      - "auto-pr"
      - "requires-review"
  
  # Validación de cambios
  validation:
    check_syntax: true
    run_linter: true
    run_tests: true
    max_code_churn: 20  # % máximo de cambio
```

### Paso 3: Script de Test GitHub

```bash
#!/bin/bash
# scripts/test-github.sh

GITHUB_TOKEN=$(cat secrets/github_token.txt)
REPO=${1:-myusername/test-repo}

echo "🧪 Testeando GitHub Connection..."

# Test 1: Authenticate
echo "1️⃣ Autenticación..."
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/user | jq '.login'

# Test 2: List repos
echo "2️⃣ Listando repositorios..."
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/user/repos?type=owner | jq '.[].name'

# Test 3: Create draft PR
echo "3️⃣ Creando PR de prueba..."
curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/$REPO/pulls \
  -d '{
    "title": "[TEST] AI Generated - Please ignore",
    "head": "test-branch",
    "base": "develop",
    "draft": true,
    "body": "This is a test PR"
  }' | jq '.url'

echo "✅ GitHub integration test complete"
```

---

## 📱 Moltbook Integration

### Paso 1: Setup en Moltbook

```bash
# En https://moltbook.com/settings/api

# Crear API Key con permisos:
- messages.read
- messages.write
- channels.list
- threads.list

# Copiar el token
```

### Paso 2: Configurar en OpenClaw

```yaml
# config/openclaw.yaml

moltbook:
  enabled: true
  
  # API Key via secret
  api_key_path: /run/secrets/moltbook_api_key
  base_url: https://api.moltbook.com
  
  # Settings
  settings:
    # Enviar updates sobre avance
    send_updates: true
    update_interval: 3600  # cada hora
    
    # Canales donde puede enviar
    allowed_channels:
      - general
      - ai-assistant
      # NO: private channels, importante
    
    # Rate limiting
    max_messages_per_hour: 10
    
    # Contenido
    message_template: |
      🤖 **OpenClaw Status Update**
      
      Tasks completed: {{completed}}
      Current status: {{status}}
      Issues: {{issues}}
      
      Last updated: {{timestamp}}
    
    # Mencionarse a sí mismo
    personality:
      name: "OpenClaw"
      emoji: "🤖"
      greeting: "¡Hola! Soy tu asistente IA para crear código"
```

### Paso 3: Script de Test Moltbook

```bash
#!/bin/bash
# scripts/test-moltbook.sh

API_KEY=$(cat secrets/moltbook_api_key.txt)
BASE_URL="https://api.moltbook.com"

echo "🧪 Testeando Moltbook Connection..."

# Test 1: Authenticate
echo "1️⃣ Autenticación..."
curl -s -H "Authorization: Bearer $API_KEY" \
  $BASE_URL/v1/user | jq '.name'

# Test 2: List channels
echo "2️⃣ Listando canales..."
curl -s -H "Authorization: Bearer $API_KEY" \
  $BASE_URL/v1/channels | jq '.[].name'

# Test 3: Send test message
echo "3️⃣ Enviando mensaje de prueba..."
curl -s -X POST \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  $BASE_URL/v1/messages \
  -d '{
    "channel": "general",
    "content": "🤖 OpenClaw status check - online and ready!"
  }' | jq '.id'

echo "✅ Moltbook integration test complete"
```

---

## 💰 Crypto Wallet Integration

### Paso 1: Setup Wallet

```bash
#!/bin/bash
# scripts/setup-wallet.sh

echo "🔐 Wallet Setup"

# Opción 1: Usar Solana (recomendado para este caso)
if ! command -v solana &> /dev/null; then
    echo "Instalando Solana CLI..."
    sh -c "$(curl -sSfL https://release.solana.com/v1.18.0/install)"
    source ~/.bashrc
fi

# Opción 2: Importar key existente
echo "¿Importar wallet existente? (y/n)"
read -r IMPORT_EXISTING

if [ "$IMPORT_EXISTING" = "y" ]; then
    echo "Pega tu private key (base58):"
    read -s PRIVATE_KEY
    
    # Guardar (será encriptado)
    echo "$PRIVATE_KEY" > /tmp/key.txt
else
    echo "Generando nueva wallet..."
    solana-keygen new --force
    WALLET_ADDRESS=$(solana address)
    echo "✅ Nueva wallet creada: $WALLET_ADDRESS"
fi
```

### Paso 2: Configurar en OpenClaw

```yaml
# config/openclaw.yaml

wallet:
  enabled: true
  
  network: devnet  # o mainnet-beta
  rpc_endpoint: https://api.devnet.solana.com
  
  # Key encriptada en secret
  private_key_path: /run/secrets/wallet_private_key
  private_key_password_path: /run/secrets/wallet_password
  
  # Seguridad
  security:
    # Solo ciertas direcciones pueden transferencias
    whitelist_recipients:
      - your-personal-wallet-address
    
    # Límites de transferencia
    max_transfer_per_tx: 10  # SOL o tokens
    max_transfers_per_day: 5
    min_balance_reserve: 1   # Mantener mínimo en wallet
  
  # Tipos de monedas soportadas
  supported_tokens:
    - token: SOL
      decimals: 9
    
    - token: USDC  # Stablecoin para más estable
      mint: EPjFWaLb3jqarzXkwiB4D5ebzr4T6W3gH1nKdTJfmts
      decimals: 6
    
    - token: MEME  # Tu memecoin
      mint: your-memecoin-mint-address
      decimals: 6
```

### Paso 3: Enviar Dinero al Agente

```typescript
// Función para transferir dinero (TypeScript)

interface TransferConfig {
  recipient: string;  // Wallet address del agente
  amount: number;
  token: 'SOL' | 'USDC' | 'MEME';
}

async function transferToAgent(config: TransferConfig) {
  // Usar Phantom, Magic Eden, o SDK directo
  
  const connection = new Connection(
    'https://api.devnet.solana.com',
    'confirmed'
  );
  
  const transaction = new Transaction().add(
    SystemProgram.transfer({
      fromPubkey: senderPublicKey,
      toPubkey: new PublicKey(config.recipient),
      lamports: config.amount * LAMPORTS_PER_SOL,
    })
  );
  
  const signature = await sendAndConfirmTransaction(
    connection,
    transaction,
    [senderKeypair]
  );
  
  return signature;
}
```

### Paso 4: Agente Verifica Balance

```typescript
// En OpenClaw agent tools

interface WalletStatus {
  address: string;
  balance: number;
  token: string;
  lastReceived?: {
    amount: number;
    from: string;
    timestamp: Date;
  };
}

async function checkWalletStatus(): Promise<WalletStatus> {
  const connection = new Connection(
    process.env.RPC_ENDPOINT,
    'confirmed'
  );
  
  const publicKey = new PublicKey(
    process.env.WALLET_ADDRESS
  );
  
  const balance = await connection.getBalance(publicKey);
  
  return {
    address: process.env.WALLET_ADDRESS,
    balance: balance / LAMPORTS_PER_SOL,
    token: 'SOL',
    lastReceived: await getLastTransactionReceived()
  };
}
```

---

## 🔍 Auditoría de PRs

### Sistema de Audit Completo

```yaml
# config/openclaw.yaml

audit:
  enabled: true
  
  # Qué registrar
  log_all:
    - pr_created
    - commits_pushed
    - files_changed
    - code_executed
    - money_received
  
  # Almacenamiento de logs
  storage:
    type: file
    path: /home/openclaw/.openclaw/logs/audit.jsonl
    rotation: daily
    retention_days: 90
  
  # Webhooks para alertas críticas
  webhooks:
    on_pr_created:
      - https://your-domain.com/webhooks/pr-created
    on_wallet_transfer:
      - https://your-domain.com/webhooks/wallet-update
    on_error:
      - https://your-domain.com/webhooks/error
```

### Estructura de Audit Log

```typescript
// Cada evento se registra así:
{
  timestamp: "2026-02-01T10:30:00Z",
  event_type: "pr_created",
  actor: "openclaw-agent",
  
  details: {
    repository: "myusername/my-project",
    pr_number: 42,
    title: "feat: Add new API endpoint",
    branch: "feature/new-endpoint",
    files_changed: 3,
    insertions: 150,
    deletions: 20,
    
    // Contexto
    task: "Create new endpoint for user authentication",
    task_id: "task-2026-001",
    
    // Validaciones
    passed_tests: true,
    lint_passed: true,
    code_review_required: true,
  },
  
  // Metadata de seguridad
  security: {
    signed_with: "agent-key-id",
    execution_sandbox: "docker-container-id",
    resource_usage: {
      cpu: "45%",
      memory: "250MB",
      time: "23s"
    }
  }
}
```

### Dashboard de Auditoría (Optional)

```bash
# Script simple para revisar logs
#!/bin/bash
# scripts/audit-review.sh

echo "📋 Audit Log Review"
echo "=================="

# Últimos eventos
echo -e "\n🔔 Últimos 10 eventos:"
tail -n 10 data/logs/audit.jsonl | jq '.'

# PRs creadas hoy
echo -e "\n📝 PRs creadas hoy:"
grep "pr_created" data/logs/audit.jsonl | \
  grep "$(date +%Y-%m-%d)" | \
  jq '.details | {title, repository, files_changed}'

# Dinero recibido
echo -e "\n💰 Transacciones:"
grep "wallet_transfer" data/logs/audit.jsonl | \
  jq '.details | {amount, token, from}'

# Errores
echo -e "\n⚠️ Errores:"
grep "error" data/logs/audit.jsonl | tail -n 5 | jq '.details'
```

---

## 🛡️ Security Hardening

### 1. Network Security

```bash
# En tu VPS

# Firewall - Solo lo necesario
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp          # SSH
sudo ufw allow 80/tcp          # HTTP (webhooks)
sudo ufw allow 443/tcp         # HTTPS (webhooks)
sudo ufw enable

# Rate limiting
sudo apt install fail2ban
sudo systemctl enable fail2ban

# DDoS protection
sudo apt install ddos-deflate
```

### 2. Container Security

```yaml
# En docker-compose.yml (ya incluido arriba)

openclaw:
  security_opt:
    - no-new-privileges:true
  cap_drop:
    - ALL
  cap_add:
    - NET_BIND_SERVICE
  read_only: true
  user: 1000:1000
```

### 3. Secret Rotation

```bash
#!/bin/bash
# scripts/rotate-secrets.sh

echo "🔄 Rotando secrets..."

# Función para rotar un secret
rotate_secret() {
  local secret_name=$1
  local secret_path=$2
  
  echo "Rotando $secret_name..."
  
  # Backup antiguo
  cp "$secret_path" "$secret_path.backup.$(date +%s)"
  
  # Nuevo valor
  echo "Ingresa nuevo valor:"
  read -s new_value
  echo "$new_value" > "$secret_path"
  chmod 600 "$secret_path"
  
  echo "✅ $secret_name rotado"
}

# Rotar cada secret
rotate_secret "GitHub Token" "secrets/github_token.txt"
rotate_secret "Gemini Key" "secrets/gemini_key.txt"
rotate_secret "Moltbook Key" "secrets/moltbook_api_key.txt"

# Reiniciar container
docker-compose restart openclaw

echo "✅ Secrets rotados y servicio reiniciado"
```

### 4. Encriptación de Datos Sensibles

```bash
# Encrypt wallet key
WALLET_PASSWORD="super-strong-password"
openssl enc -aes-256-cbc -salt \
  -in wallet_private_key.txt \
  -out secrets/wallet_private_key.enc \
  -k "$WALLET_PASSWORD"

# Decrypt cuando sea necesario
openssl enc -d -aes-256-cbc \
  -in secrets/wallet_private_key.enc \
  -k "$WALLET_PASSWORD"
```

---

## 📊 Monitoreo

### 1. Health Check

```bash
#!/bin/bash
# scripts/health-check.sh

echo "🏥 Health Check"

# 1. Docker container
if docker ps | grep -q openclaw; then
  echo "Container: ✅ running"
else
  echo "Container: ❌ not running"
fi

# 2. API
if curl -sf http://localhost:18789/health > /dev/null; then
  echo "API: ✅ healthy"
else
  echo "API: ❌ unavailable"
fi

# 3. GitHub access
GITHUB_USER=$(curl -sf -H "Authorization: token $(cat secrets/github_token.txt)" \
  https://api.github.com/user | jq -r '.login')
if [ -n "$GITHUB_USER" ]; then
  echo "GitHub: ✅ $GITHUB_USER"
else
  echo "GitHub: ❌ auth failed"
fi

# 4. Wallet balance (if solana CLI available)
if command -v solana &> /dev/null; then
  echo "Wallet balance: $(solana balance 2>/dev/null || echo 'N/A')"
fi

# 5. Logs de error
echo -e "\n📋 Recent errors:"
grep -i error data/logs/*.log 2>/dev/null | tail -5 || echo "No errors found"
```

### 2. Alerts & Monitoring

```yaml
# config/openclaw.yaml

monitoring:
  enabled: true
  
  alerts:
    - condition: "cpu_usage > 80%"
      action: "log_warning"
      webhook: "https://your-domain.com/alerts"
    
    - condition: "memory_usage > 900MB"
      action: "log_warning"
    
    - condition: "pr_creation_failed"
      action: "email_notify"
      recipients:
        - "your@email.com"
    
    - condition: "wallet_balance_low < 0.1 SOL"
      action: "moltbook_message"
      message: "⚠️ Wallet balance running low"
    
    - condition: "error_rate > 5%"
      action: "restart_container"
```

### 3. Logs Centralizados (Optional)

```bash
# Enviar logs a un servicio como Datadog o ELK

docker-compose.yml adicional:

  logstash:
    image: docker.elastic.co/logstash/logstash:8.0.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    environment:
      - PIPELINE_WORKERS=2
```

---

## 🧪 Testing & Deployment

### 1. Pre-Deployment Checklist

```bash
#!/bin/bash
# scripts/pre-deploy.sh

echo "✅ Pre-Deployment Checklist"

# 1. Secrets
test -f secrets/github_token.txt && echo "✅ GitHub token" || exit 1
test -f secrets/gemini_key.txt && echo "✅ Gemini key" || exit 1
test -f secrets/moltbook_api_key.txt && echo "✅ Moltbook key" || exit 1
test -f secrets/wallet_private_key.enc && echo "✅ Wallet key" || exit 1

# 2. Config
test -f config/openclaw.yaml && echo "✅ Config file" || exit 1

# 3. Docker
docker-compose config > /dev/null && echo "✅ Docker compose" || exit 1

# 4. Tests
./scripts/test-github.sh && echo "✅ GitHub" || exit 1
./scripts/test-moltbook.sh && echo "✅ Moltbook" || exit 1
./scripts/test-wallet.sh && echo "✅ Wallet" || exit 1

# 5. Security scan
trivy image openclaw:latest

echo -e "\n✅ Todos los checks pasaron. Listo para deploy!"
```

### 2. Deploy Script

```bash
#!/bin/bash
# scripts/deploy.sh

set -e

echo "🚀 Deploying OpenClaw..."

# 1. Pre-checks
./scripts/pre-deploy.sh

# 2. Build image
echo "🔨 Building Docker image..."
docker-compose build

# 3. Pull & update
echo "📥 Pulling latest code..."
docker-compose pull

# 4. Start services
echo "🟢 Starting services..."
docker-compose up -d

# 5. Wait for health
sleep 10
curl -f http://localhost:18789/health || exit 1

# 6. Backup
echo "💾 Backing up data..."
./scripts/backup.sh

echo "✅ Deploy complete!"
docker-compose ps
```

### 3. Backup Script

```bash
#!/bin/bash
# scripts/backup.sh

BACKUP_DIR="./backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

mkdir -p "$BACKUP_DIR"

echo "💾 Backing up data..."

# Backup sessions
tar -czf "$BACKUP_DIR/sessions_$TIMESTAMP.tar.gz" data/sessions/

# Backup logs
tar -czf "$BACKUP_DIR/logs_$TIMESTAMP.tar.gz" data/logs/

# Backup config
tar -czf "$BACKUP_DIR/config_$TIMESTAMP.tar.gz" config/

# Keep only last 7 days
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +7 -delete

echo "✅ Backup complete. Files in: $BACKUP_DIR"
```

---

## 🚦 Primeros Pasos (Quick Start)

### 1. Setup Completo (30 min)

```bash
# En tu PC
mkdir openclaw-setup && cd openclaw-setup

# Copiar archivos (docker-compose.yml, Dockerfile, etc.)
# de: ~/openclaw-setup/

# 2. SSH a VPS y preparar
ssh openclaw@your_vps_ip

cd ~/openclaw-setup
chmod +x scripts/*.sh

# 3. Crear secrets
./scripts/setup-secrets.sh

# 4. Build & deploy
docker-compose build
docker-compose up -d

# 5. Verificar
docker-compose logs openclaw
./scripts/health-check.sh
```

### 2. Primeras Pruebas

```bash
# Test GitHub
./scripts/test-github.sh

# Test Moltbook
./scripts/test-moltbook.sh

# Test Wallet
./scripts/test-wallet.sh

# Ver logs
docker-compose logs openclaw -f
```

### 3. Primer Task al Agente

```bash
# Enviar mensaje vía Moltbook
# Message: "Create a simple Node.js hello-world app and make a PR to my-project/develop"

# El agente debería:
# 1. Generar código
# 2. Crear rama
# 3. Hacer commit
# 4. Crear PR en GitHub
# 5. Notificar en Moltbook
```

---

## ⚠️ Troubleshooting

### Problema: "Permission denied" en secrets

```bash
chmod 600 secrets/*
docker-compose down
docker-compose up -d
```

### Problema: "GitHub API rate limit"

```bash
# Aumenta el token scope o usa GitHub App (no token personal)
# Implementa caching de requests
```

### Problema: "Wallet balance insufficient"

```bash
# Enviar dinero a wallet address
solana address  # ver address
# Transferir desde otra cuenta
```

### Problema: "Container keeps restarting"

```bash
# Check logs
docker-compose logs openclaw

# Rebuild
docker-compose down
docker-compose build --no-cache
docker-compose up -d
```

### Problema: "Moltbook no recibe mensajes"

```bash
# Verificar API key
./scripts/test-moltbook.sh

# Verificar allowed_channels en config
cat config/openclaw.yaml | grep allowed_channels
```

---

## 📈 Métricas & Observabilidad

```yaml
# Qué monitorear

metrics:
  - pr_created_per_day
  - pr_merge_rate
  - code_quality_score
  - test_pass_rate
  - api_latency
  - wallet_balance
  - error_rate
  - uptime
  - resource_usage
    - cpu
    - memory
    - disk
```

---

## 🎯 Guía de Mantenimiento

### Diario
- [ ] Revisar logs de errores
- [ ] Verificar wallet balance
- [ ] Revisar PRs creadas
- [ ] Health check

### Semanal
- [ ] Revisar metrics
- [ ] Actualizar OpenClaw
- [ ] Revisar GitHub issues

### Mensual
- [ ] Rotar secrets (GitHub token)
- [ ] Backup & recovery test
- [ ] Security audit
- [ ] Performance tuning

---

## 💡 Tips & Tricks

### 1. SSH Config

```bash
# ~/.ssh/config

Host openclaw-vps
    HostName your_vps_ip
    User openclaw
    IdentityFile ~/.ssh/openclaw_vps
    StrictHostKeyChecking accept-new
```

```bash
# Uso
ssh openclaw-vps
```

### 2. Local Development

```bash
# Probar localmente antes de VPS
docker-compose -f docker-compose.local.yml up

# Usar secretos dummy para dev
echo "dummy-token" > secrets/github_token.txt
```

### 3. Emergency Access

```bash
# Si algo falla, acceso directo al container
docker-compose exec openclaw bash

# O como root
sudo docker exec -it openclaw-agent bash
```

---

## 🔗 Referencias

- [OpenClaw Docs](https://docs.openclaw.ai)
- [GitHub API](https://docs.github.com/en/rest)
- [Solana Docs](https://docs.solana.com)
- [Docker Compose](https://docs.docker.com/compose/)

---

## 📞 Soporte

Si algo falla:
1. Revisar logs: `docker-compose logs openclaw`
2. Ejecutar health check: `./scripts/health-check.sh`
3. Revisar troubleshooting arriba
4. Abrir issue en GitHub

---

**Última actualización**: Febrero 1, 2026  
**Versión**: 1.0  
**Status**: Production-ready ✅

