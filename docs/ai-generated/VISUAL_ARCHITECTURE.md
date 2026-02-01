# OpenClaw - Diagramas y Visualización de Arquitectura

## 📊 Árbol de Dependencias del Core

```
┌─────────────────────────────────────────────────────────────────┐
│                          openclaw CLI                           │
│                      (src/cli/program.ts)                       │
└────────────────────────────────┬────────────────────────────────┘
         │
         ├─── gateway command
         ├─── agent command
         ├─── message send command
         ├─── channels command
         ├─── plugins command
         ├─── memory command
         ├─── models command
         ├─── onboard command
         └─── ... (~50+ commands)
         
         ↓

┌─────────────────────────────────────────────────────────────────┐
│                        CLI Dependencies                         │
│                       (src/cli/deps.ts)                         │
└────────────────────────────────┬────────────────────────────────┘
         │
         ├─── Config Loader (src/config/)
         ├─── Logger (src/logging/)
         ├─── Channel Registry (src/channels/registry.ts)
         ├─── Provider Registry (src/providers/)
         ├─── Session Manager (src/gateway/session-utils.ts)
         └─── Agent Dispatcher (src/agents/)
```

---

## 🔄 Data Flow Diagram

```
External Channels
(WhatsApp, Telegram, etc)
         │
         ▼
┌──────────────────────────┐
│  Channel Adapters        │
│  (src/channels/*.ts)     │
└────────┬─────────────────┘
         │ (normalize)
         ▼
┌──────────────────────────┐
│  Gateway Registry        │
│  (channel-config.ts)     │
└────────┬─────────────────┘
         │ (route)
         ▼
┌──────────────────────────┐
│  Message Router          │
│  (routing/targets.ts)    │
└────────┬─────────────────┘
         │ (validate)
         ▼
┌──────────────────────────┐
│  Validation Layer        │
│  - Allowlists            │
│  - Command Gating        │
│  - Permissions           │
└────────┬─────────────────┘
         │ (dispatch)
         ▼
┌──────────────────────────────────────────┐
│        Agent Engine (src/agents/)        │
│                                          │
│  ┌────────────────────────────────────┐ │
│  │ 1. Load/Create Session             │ │
│  │ 2. Build System Prompt             │ │
│  │ 3. Prepare Tools                   │ │
│  │ 4. Run Pi Embedded Agent           │ │
│  │ 5. Execute Tool Calls              │ │
│  │ 6. Stream Response                 │ │
│  │ 7. Save to Memory                  │ │
│  └────────────────────────────────────┘ │
└────────┬───────────────────────────────┘
         │
         ├─────────────────────┬────────────────────────┐
         │                     │                        │
         ▼                     ▼                        ▼
    ┌────────────┐      ┌──────────────┐      ┌──────────────┐
    │ Response   │      │ Memory Save  │      │ Analytics    │
    │ Formatting │      │ (vectors)    │      │ & Logging    │
    └────┬───────┘      └──────────────┘      └──────────────┘
         │
         ▼
┌──────────────────────────┐
│  Channel Senders         │
│  (send response)         │
└────────┬─────────────────┘
         │
         ▼
External Channels
(WhatsApp, Telegram, etc)
```

---

## 🏗️ Module Organization

```
src/
│
├── cli/                          ← CLI Layer (10 commands)
│   ├── program.ts               ← Main router
│   ├── deps.ts                  ← DI container
│   ├── gateway-cli.ts
│   ├── agent.ts
│   └── ...
│
├── gateway/                      ← Server Layer (60+ files)
│   ├── server.impl.ts           ← Main server
│   ├── server-chat.ts           ← Chat handling
│   ├── server-channels.ts       ← Channel management
│   ├── server-plugins.ts        ← Plugin system
│   ├── auth.ts                  ← Authentication
│   ├── protocol/                ← Communication
│   └── ...
│
├── agents/                       ← AI Engine (150+ files)
│   ├── pi-embedded.ts           ← Core agent
│   ├── pi-embedded-runner.ts    ← Executor
│   ├── pi-embedded-subscribe.ts ← Event streaming
│   ├── auth-profiles/           ← Auth management
│   ├── model-catalog.ts         ← Model registry
│   ├── model-failover.ts        ← Failover logic
│   ├── tools/                   ← Tool definitions
│   ├── sandbox/                 ← Sandbox execution
│   └── ... (many helper modules)
│
├── channels/                     ← Messaging Layer (50+ files)
│   ├── registry.ts              ← Channel registry
│   ├── channel-config.ts        ← Configuration
│   ├── command-gating.ts        ← Access control
│   ├── allowlists/              ← Allow/deny lists
│   ├── plugins/                 ← Plugin integration
│   ├── web/                     ← Web UI
│   └── ... (core channel adapters)
│
├── providers/                    ← Model Providers (10+ files)
│   ├── model-catalog.ts
│   ├── model-selection.ts
│   ├── model-failover.ts
│   ├── model-auth.ts
│   └── ...
│
├── routing/                      ← Routing Logic (10+ files)
│   ├── targets.ts
│   ├── allowlists/
│   └── ...
│
├── memory/                       ← Persistent Storage (20+ files)
│   ├── index.ts
│   ├── search.ts
│   └── ...
│
├── media/                        ← Media Processing (30+ files)
│   ├── image.ts
│   ├── video.ts
│   ├── pdf.ts
│   └── ...
│
├── browser/                      ← Browser Automation (20+ files)
│   ├── playwright integration
│   └── ...
│
├── tts/                          ← Text-to-Speech (5+ files)
│   └── edge-tts integration
│
├── config/                       ← Configuration (10+ files)
│   └── config.ts
│
├── logging/                      ← Logging System (5+ files)
│   └── structured logging
│
├── infra/                        ← Infrastructure (15+ files)
│   ├── runtime-guard.ts
│   ├── env.ts
│   ├── errors.ts
│   └── ...
│
├── security/                     ← Security Layer (10+ files)
│   ├── auth.ts
│   └── ...
│
├── terminal/                     ← TUI Layer (20+ files)
│   ├── palette.ts
│   ├── table.ts
│   └── ...
│
├── sessions/                     ← Session Management
│   └── session persistence, context
│
├── pairing/                      ← Device Pairing
│   └── QR code pairing, auth
│
├── wizard/                       ← Onboarding Wizard
│   └── onboarding flows
│
├── daemon/                       ← Background Service
│   └── daemon mode support
│
├── cron/                         ← Scheduled Tasks
│   └── cron job management
│
├── hooks/                        ← Lifecycle Hooks
│   └── event hooks system
│
├── utils/                        ← Utilities (30+ files)
│   └── helper functions
│
└── test-helpers/                 ← Test Utilities
    └── mocking, fixtures
```

---

## 🔌 Extension Architecture

```
extensions/
│
├── bluebubbles/                  ← Channel Extension
│   ├── package.json
│   ├── src/index.ts             ← Plugin entry
│   ├── src/channel.ts           ← Implementation
│   └── dist/
│
├── msteams/                      ← Channel Extension
├── matrix/                       ← Channel Extension
├── voice-call/                   ← Feature Extension
├── memory-lancedb/               ← Storage Extension
├── google-antigravity-auth/      ← Auth Extension
│
└── ... (~30+ more extensions)
```

**Plugin Loading Flow:**
```
src/plugins/ (plugin registry)
    ↓
Auto-discover extensions/*/
    ↓
Load plugin manifest
    ↓
Call plugin.initialize()
    ↓
Register channels/models/tools
    ↓
Plugin active
```

---

## 💾 Data Storage Architecture

```
~/.openclaw/
│
├── config/
│   ├── config.yaml              ← Main config
│   ├── models.json              ← Model config
│   └── channels.yaml
│
├── credentials/
│   ├── encrypted_keys/          ← Secure storage
│   ├── oauth_tokens/
│   └── api_keys/
│
├── sessions/
│   ├── <session-id>/
│   │   ├── metadata.json
│   │   ├── history.jsonl        ← Conversation history
│   │   ├── messages/            ← Raw messages
│   │   └── attachments/
│   └── ...
│
├── memory/
│   ├── vectors.db               ← LanceDB (embeddings)
│   ├── metadata.db              ← SQLite (metadata)
│   └── cache/
│
├── logs/
│   ├── openclaw.log             ← Main log
│   ├── gateway.log
│   ├── agents.log
│   └── ...
│
└── cache/
    ├── models/
    ├── config/
    └── ...
```

---

## 🔐 Security & Auth Flow

```
┌──────────────────────────┐
│   User Credentials       │
│ (OAuth / API Keys)       │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│  Credential Encryption   │
│  (secure storage)        │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│  Auth Profile Selection  │
│  (failover strategy)     │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│  Model API Call          │
│  (with selected profile) │
└────────────┬─────────────┘
             │
      ┌──────┴──────┐
      │             │
   Success      Failure
      │             │
      ▼             ▼
   Cache      Mark Failed
      │          │
      │          ▼
      │      Cooldown
      │          │
      │          ▼
      │      Retry with
      │      Next Profile
      │
      ▼
   Response
```

---

## 📈 Scaling & Performance Layers

```
┌─────────────────────────────────────────┐
│      L1: Memory Cache (1-5s TTL)        │
│      (fast, local, in-process)          │
└─────────────────────────────────────────┘
              ↓ cache miss
┌─────────────────────────────────────────┐
│     L2: Disk Cache (1-1h TTL)           │
│      (sqlite, filesystem)                │
└─────────────────────────────────────────┘
              ↓ cache miss
┌─────────────────────────────────────────┐
│    L3: Remote Cache (Redis if used)     │
│      (cluster, distributed)              │
└─────────────────────────────────────────┘
              ↓ cache miss
┌─────────────────────────────────────────┐
│         L4: Live Data Fetch             │
│      (API call, expensive)               │
└─────────────────────────────────────────┘
```

---

## 🧬 Test Coverage Architecture

```
Unit Tests (70%)
├── src/**/*.test.ts
│   ├── Component tests
│   ├── Function tests
│   ├── Utility tests
│   └── Mock-based
│
├── Coverage Targets:
│   ├── Lines: 70%
│   ├── Branches: 70%
│   ├── Functions: 70%
│   └── Statements: 70%
│
└── Tools:
    ├── Vitest (test runner)
    ├── V8 (coverage provider)
    └── Mocking library
       
        │
        ▼

E2E Tests (selected features)
├── src/**/*.e2e.test.ts
│   ├── Full flow tests
│   ├── Integration tests
│   ├── Docker-based
│   └── Real services
│
└── Tools:
    ├── Vitest
    └── Docker compose
       
        │
        ▼

Live Tests (on-demand)
├── LIVE=1 pnpm test:live
│   ├── Real API calls
│   ├── Production models
│   ├── Full integration
│   └── Long-running
│
└── Tools:
    ├── Vitest
    ├── Real credentials
    └── Rate limiting
```

---

## 🔄 Build & Release Pipeline

```
Source Code
├── TypeScript (.ts files)
├── Swift (.swift files)
├── Kotlin (.kt files)
└── Other assets
         │
         ▼
┌──────────────────────────┐
│    Pre-commit Checks     │
│  ├─ Lint (oxlint)        │
│  ├─ Format (oxfmt)       │
│  ├─ Type check           │
│  └─ Tests                │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│    Compilation           │
│  ├─ tsc (TypeScript)     │
│  ├─ Swift compiler       │
│  ├─ Kotlin compiler      │
│  └─ Assets copying       │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│    Build Artifacts       │
│  ├─ dist/ (JS)           │
│  ├─ OpenClaw.app (macOS) │
│  ├─ APK (Android)        │
│  └─ IPA (iOS)            │
└────────────┬─────────────┘
             │
             ├─────────────────────┬──────────────────┐
             │                     │                  │
             ▼                     ▼                  ▼
        ┌─────────┐          ┌──────────┐      ┌────────────┐
        │ npm pkg │          │ Mac App  │      │Mobile Apps │
        │ publish │          │ dmg dist │      │  app store │
        └─────────┘          └──────────┘      └────────────┘
             │                     │                  │
             └─────────────────────┴──────────────────┘
                           │
                           ▼
                    Release (stable/beta/dev)
```

---

## 📊 Component Interaction Matrix

```
                CLI  GW   AG   CH   PR   MEM  TOL  CFG
CLI             -    ✓    ✓    ✓    ✓    ✓    ✓    ✓
GW              ✓    -    ✓    ✓    ✓    ✓    ✓    ✓
AG              ✓    ✓    -    ✓    ✓    ✓    ✓    ✓
CH              ✓    ✓    ✓    -    ✓    ✓    ✓    ✓
PR              ✓    ✓    ✓    ✓    -    ✓    ✓    ✓
MEM             ✓    ✓    ✓    ✓    ✓    -    ✓    ✓
TOL             ✓    ✓    ✓    ✓    ✓    ✓    -    ✓
CFG             ✓    ✓    ✓    ✓    ✓    ✓    ✓    -

Legend:
CLI = CLI Module
GW  = Gateway
AG  = Agents
CH  = Channels
PR  = Providers
MEM = Memory
TOL = Tools
CFG = Config

✓ = Direct dependency
- = No dependency
```

---

## 🎯 Decision Tree - Dónde Agregar Código

```
¿Qué necesitas agregar?
│
├─→ Nuevo Canal de Mensajería
│   └─→ src/channels/
│       o extensions/canal-name/
│
├─→ Nuevo Proveedor de Modelo
│   └─→ src/providers/
│       o crear extension
│
├─→ Nueva Herramienta para Agente
│   └─→ src/agents/tools/
│
├─→ Nuevo Comando CLI
│   └─→ src/cli/
│       + registrar en program.ts
│
├─→ Nueva Feature en Gateway
│   └─→ src/gateway/
│       + server-*.ts
│
├─→ Lógica de Almacenamiento
│   └─→ src/memory/
│
├─→ Procesamiento Multimedia
│   └─→ src/media/
│
├─→ Automatización Web
│   └─→ src/browser/
│
├─→ Enrutamiento/Lógica
│   └─→ src/routing/
│
├─→ Utilidades Compartidas
│   └─→ src/utils/
│
└─→ Plugin/Extensión
    └─→ extensions/nombre/
```

---

## 📉 Complexity by Module

```
src/agents/              ████████████████████░░░░░░░ (HIGHEST)
  - 150+ archivos
  - Lógica compleja de IA
  - Manejo de tools y streaming
  - Compresión de contexto

src/gateway/             ██████████████░░░░░░░░░░░░░ (HIGH)
  - 60+ archivos
  - WebSocket server
  - Múltiples protocolos
  - Gestión de sesiones

src/channels/            ███████████░░░░░░░░░░░░░░░░ (MEDIUM-HIGH)
  - 50+ archivos
  - Múltiples adaptadores
  - Protocolos variados

src/providers/           ███████░░░░░░░░░░░░░░░░░░░░ (MEDIUM)
  - 10+ archivos
  - Selección de modelos
  - Failover logic

src/routing/             ██████░░░░░░░░░░░░░░░░░░░░░ (MEDIUM)
  - 10+ archivos
  - Enrutamiento inteligente

src/memory/              █████░░░░░░░░░░░░░░░░░░░░░░ (MEDIUM)
  - 20+ archivos
  - Búsqueda vectorial

src/cli/                 █████░░░░░░░░░░░░░░░░░░░░░░ (MEDIUM)
  - 50+ archivos
  - 50+ comandos

src/infra/               ████░░░░░░░░░░░░░░░░░░░░░░░ (MEDIUM-LOW)
  - 15+ archivos
  - Utilities

src/utils/               ███░░░░░░░░░░░░░░░░░░░░░░░░ (LOW)
  - Helper functions
```

---

## 🎓 Learning Path

```
Start
  ↓
[1] Understand CLI Entry
    src/index.ts
    src/cli/program.ts
  ↓
[2] Learn Gateway Basics
    src/gateway/server.impl.ts
    src/gateway/server-chat.ts
  ↓
[3] Understand Agent System
    src/agents/pi-embedded.ts
    src/agents/pi-embedded-runner.ts
  ↓
[4] Learn Channels
    src/channels/registry.ts
    Pick one: telegram, slack, discord
  ↓
[5] Deep Dive: Models & Auth
    src/agents/auth-profiles/
    src/providers/model-catalog.ts
  ↓
[6] Advanced: Tools & Sandbox
    src/agents/tools/
    src/agents/sandbox/
  ↓
[7] Integration: Memory & Storage
    src/memory/
    src/sessions/
  ↓
Proficient! ✓
```

---

## 📝 Resumen de Estadísticas

```
Project OpenClaw - Statistics

├─ Language Mix
│  ├─ TypeScript:  85% (src/, tests)
│  ├─ Swift:        5% (apps/macos, apps/ios)
│  ├─ Kotlin:       5% (apps/android)
│  └─ Other:        5% (scripts, config)
│
├─ File Count
│  ├─ TypeScript:   500+ files
│  ├─ Tests:        200+ test files
│  ├─ Swift:        100+ files
│  ├─ Kotlin:        50+ files
│  └─ Config:        30+ files
│
├─ Lines of Code (approx)
│  ├─ Source:      100,000+
│  ├─ Tests:        50,000+
│  ├─ Docs:         30,000+
│  └─ Total:       180,000+
│
├─ Module Breakdown
│  ├─ src/agents/:    15,000+ LOC
│  ├─ src/gateway/:   12,000+ LOC
│  ├─ src/channels/:  10,000+ LOC
│  ├─ src/cli/:        8,000+ LOC
│  └─ Others:        55,000+ LOC
│
└─ Dependencies
   ├─ Core deps:      30+
   ├─ Dev deps:       15+
   ├─ Peer deps:       5+
   └─ Total:          50+
```

---

## 🔗 Quick Links to Key Files

| Concepto | Archivo | Líneas |
|----------|---------|--------|
| Entry Point | `src/index.ts` | 50 |
| CLI Router | `src/cli/program.ts` | 200+ |
| Gateway Server | `src/gateway/server.impl.ts` | 500+ |
| Agent Core | `src/agents/pi-embedded.ts` | 300+ |
| Channel Registry | `src/channels/registry.ts` | 150+ |
| Config Loading | `src/config/config.ts` | 100+ |
| Memory Search | `src/memory/index.ts` | 200+ |
| Tool Execution | `src/agents/tools/` | 500+ |
| Test Setup | `vitest.config.ts` | 50 |
| Package Manager | `package.json` | 277 |

---

Este diagrama proporciona una vista visual completa de la arquitectura de OpenClaw.

