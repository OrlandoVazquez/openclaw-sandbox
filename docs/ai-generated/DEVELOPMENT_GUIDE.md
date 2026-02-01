# OpenClaw - Guía de Desarrollo

## 🚀 Setup Inicial

### Requisitos
- **Node.js**: ≥22.12.0 (verificar con `node --version`)
- **pnpm**: 10.23.0 (package manager)
- **Git**: Para versionamiento
- **TypeScript**: 5.9.3 (incluido en dependencias)

### Instalación del Proyecto

```bash
# Clonar repositorio
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# Instalar dependencias (usa pnpm)
pnpm install

# Compilar TypeScript
pnpm build

# Ejecutar tests
pnpm test

# Iniciar en desarrollo
pnpm dev
```

---

## 📁 Estructura de Directorios para Desarrollo

```
src/
├── cli/                      # Comandos de CLI
│   ├── program.ts           # Definición de comandos
│   ├── deps.ts              # Inyección de dependencias
│   ├── gateway-cli.ts       # Comando gateway
│   ├── agent.ts             # Comando agent
│   └── ...
│
├── gateway/                  # Servidor WebSocket + HTTP
│   ├── server.impl.ts       # Implementación principal
│   ├── server-chat.ts       # Gestión de chats
│   ├── server-channels.ts   # Gestión de canales
│   ├── auth.ts              # Autenticación
│   └── protocol/            # Protocolos de comunicación
│
├── agents/                   # Motor de agentes
│   ├── pi-embedded.ts       # Core del agente
│   ├── pi-embedded-runner.ts # Ejecutor
│   ├── auth-profiles/       # Profiles de auth
│   ├── model-catalog.ts     # Catálogo de modelos
│   └── tools/               # Herramientas disponibles
│
├── channels/                 # Adaptadores de canales
│   ├── registry.ts          # Registro centralizado
│   ├── channel-config.ts    # Configuración
│   ├── plugins/             # Plugin system
│   └── web/                 # UI web
│
├── providers/                # Modelos y autenticación
│   ├── model-catalog.ts
│   ├── model-selection.ts
│   └── model-failover.ts
│
├── routing/                  # Enrutamiento
│   ├── targets.ts
│   ├── allowlists/
│   └── ...
│
├── config/                   # Gestión de configuración
│   └── config.ts
│
├── memory/                   # Almacenamiento persistente
│   ├── index.ts
│   └── ...
│
├── media/                    # Procesamiento multimedia
│   ├── image.ts
│   ├── video.ts
│   └── ...
│
├── infra/                    # Infraestructura
│   ├── runtime-guard.ts
│   ├── env.ts
│   ├── errors.ts
│   └── ...
│
└── test-helpers/             # Utilidades para tests
```

---

## 🛠️ Flujo de Desarrollo Típico

### 1. Crear una Rama de Característica

```bash
# Actualizar main
git pull origin main

# Crear rama
git checkout -b feat/nueva-caracteristica

# O usar git flow
git flow feature start nueva-caracteristica
```

### 2. Hacer Cambios

```bash
# Editar archivos TypeScript en src/
# Ejemplo: agregar nuevo comando CLI

# src/cli/programa.ts
# Agregar nuevo comando
.command('new-command')
  .description('Nueva funcionalidad')
  .action(async (options) => {
    // implementación
  });
```

### 3. Linting y Formateo

```bash
# Verificar linting
pnpm lint

# Arreglar automáticamente
pnpm lint:fix

# Formatear código
pnpm format:fix
```

### 4. Escribir Tests

```bash
# Crear archivo de test
# src/cli/nueva-caracteristica.test.ts

import { describe, it, expect, beforeEach } from 'vitest';
import { nuevoComando } from './nueva-caracteristica';

describe('Nueva Característica', () => {
  it('debe ejecutarse correctamente', async () => {
    const resultado = await nuevoComando({ ... });
    expect(resultado).toBeDefined();
  });
});

# Ejecutar tests
pnpm test

# Tests en modo watch
pnpm test:watch

# Con cobertura
pnpm test:coverage
```

### 5. Build Local

```bash
# Compilar TypeScript
pnpm build

# Verificar que no hay errores
pnpm lint

# Ejecutar CLI en desarrollo
pnpm openclaw --help
```

### 6. Commit

```bash
# Usando el script de committer (recomendado)
scripts/committer "CLI: add new-command feature" src/cli/programa.ts

# O manual (menos recomendado)
git add src/cli/programa.ts
git commit -m "CLI: add new-command feature"
```

### 7. Pull Request

```bash
# Push a la rama
git push origin feat/nueva-caracteristica

# Crear PR en GitHub
# (o usar gh CLI)
gh pr create --title "feat: Add new command" \
  --body "Description of changes"
```

---

## 🏗️ Estructura de Archivos TypeScript

### Archivo Estándar

```typescript
// src/mi-modulo/index.ts

// 1. Imports
import { something } from 'node:fs';
import { Logger } from '../logging.js';
import type { Config } from '../config/index.js';

// 2. Types/Interfaces
interface MyOptions {
  name: string;
  enabled?: boolean;
}

interface MyResult {
  success: boolean;
  data?: unknown;
}

// 3. Constants
const DEFAULT_TIMEOUT = 5000;

// 4. Main Class/Function
export class MyService {
  private logger: Logger;
  
  constructor(private config: Config) {
    this.logger = new Logger('MyService');
  }
  
  async initialize(): Promise<void> {
    this.logger.info('Initializing service');
    // ...
  }
  
  async doSomething(options: MyOptions): Promise<MyResult> {
    try {
      // Implementación
      return { success: true, data: result };
    } catch (error) {
      this.logger.error('Error in doSomething', error);
      throw error;
    }
  }
}

// 5. Exports
export { MyService };
export type { MyOptions, MyResult };
```

### Test Estándar

```typescript
// src/mi-modulo/index.test.ts

import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { MyService } from './index.js';
import type { Config } from '../config/index.js';

describe('MyService', () => {
  let service: MyService;
  let config: Config;
  
  beforeEach(() => {
    config = {
      // Mock config
    };
    service = new MyService(config);
  });
  
  afterEach(async () => {
    await service.cleanup?.();
  });
  
  it('should initialize successfully', async () => {
    await expect(service.initialize()).resolves.not.toThrow();
  });
  
  it('should handle errors gracefully', async () => {
    const result = await service.doSomething({
      name: 'test'
    });
    
    expect(result.success).toBe(true);
  });
});
```

---

## 🔌 Agregar un Nuevo Canal

### Pasos

#### 1. Crear archivo del canal

```bash
# src/channels/my-new-channel.ts
touch src/channels/my-new-channel.ts
```

#### 2. Implementar interfaz Channel

```typescript
// src/channels/my-new-channel.ts

import type { Channel, Message, SendTarget } from './types.js';

export class MyNewChannel implements Channel {
  id = 'my-new-channel';
  name = 'My New Channel';
  
  private connected = false;
  
  async connect(): Promise<void> {
    // Conectar con API/SDK del canal
    this.connected = true;
  }
  
  async disconnect(): Promise<void> {
    // Desconectar
    this.connected = false;
  }
  
  isConnected(): boolean {
    return this.connected;
  }
  
  async send(message: Message, target: SendTarget): Promise<void> {
    if (!this.connected) {
      throw new Error('Channel not connected');
    }
    
    // Enviar mensaje a través del API del canal
    await this.api.sendMessage({
      to: target.recipientId,
      text: message.text,
      attachments: message.attachments
    });
  }
  
  onMessage(listener: (msg: Message) => void): void {
    // Registrar listener para mensajes entrantes
    this.api.on('message', (data) => {
      listener({
        text: data.text,
        sender: data.from,
        channel: this.id,
        timestamp: new Date(),
        attachments: data.attachments
      });
    });
  }
  
  getIdentity() {
    return {
      channelId: this.id,
      providerId: this.api.userId
    };
  }
  
  getCapabilities() {
    return {
      supportsAttachments: true,
      supportsTypingIndicator: true,
      supportsReadReceipt: true
    };
  }
}
```

#### 3. Registrar en registry

```typescript
// src/channels/registry.ts

import { MyNewChannel } from './my-new-channel.js';

// En initialize():
const myNewChannel = new MyNewChannel(config);
this.register('my-new-channel', myNewChannel);
```

#### 4. Tests

```typescript
// src/channels/my-new-channel.test.ts

import { describe, it, expect, beforeEach, vi } from 'vitest';
import { MyNewChannel } from './my-new-channel.js';

describe('MyNewChannel', () => {
  let channel: MyNewChannel;
  
  beforeEach(() => {
    channel = new MyNewChannel();
  });
  
  it('should connect successfully', async () => {
    await expect(channel.connect()).resolves.not.toThrow();
    expect(channel.isConnected()).toBe(true);
  });
  
  it('should send message', async () => {
    await channel.connect();
    
    await expect(
      channel.send(
        { text: 'Hello', sender: 'user' },
        { recipientId: 'target' }
      )
    ).resolves.not.toThrow();
  });
});
```

---

## 🛠️ Agregar una Herramienta (Tool)

### Pasos

#### 1. Crear la herramienta

```typescript
// src/agents/tools/my-tool.ts

import type { Tool, ToolResult, ToolExecutionContext } from './types.js';

export class MyTool implements Tool {
  id = 'my_tool';
  name = 'My Custom Tool';
  description = 'Does something useful';
  
  inputSchema = {
    type: 'object',
    properties: {
      input: {
        type: 'string',
        description: 'Input parameter'
      }
    },
    required: ['input']
  };
  
  async execute(
    input: unknown,
    context: ToolExecutionContext
  ): Promise<ToolResult> {
    const { input: param } = input as { input: string };
    
    try {
      // Implementar lógica
      const result = processInput(param);
      
      return {
        type: 'text',
        content: result
      };
    } catch (error) {
      return {
        type: 'error',
        content: `Error: ${(error as Error).message}`
      };
    }
  }
}
```

#### 2. Registrar en el agente

```typescript
// src/agents/pi-tools.ts

import { MyTool } from './tools/my-tool.js';

const tools = [
  new BashTool(),
  new ScreenshotTool(),
  new MyTool(),  // Agregar aquí
  // ...
];
```

#### 3. Tests

```typescript
// src/agents/tools/my-tool.test.ts

import { describe, it, expect } from 'vitest';
import { MyTool } from './my-tool.js';

describe('MyTool', () => {
  it('should execute successfully', async () => {
    const tool = new MyTool();
    
    const result = await tool.execute(
      { input: 'test' },
      {} as any
    );
    
    expect(result.type).toBe('text');
  });
});
```

---

## 🔌 Crear un Plugin/Extensión

### Estructura

```
extensions/my-extension/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts            # Entry point
│   ├── channel.ts          # Si es canal
│   └── ...
├── dist/
└── README.md
```

### package.json

```json
{
  "name": "@openclaw/ext-my-extension",
  "version": "1.0.0",
  "description": "My custom extension",
  "type": "module",
  "main": "dist/index.js",
  "exports": {
    ".": "./dist/index.js"
  },
  "dependencies": {
    "mi-libreria": "^1.0.0"
  },
  "devDependencies": {
    "openclaw": "*",
    "typescript": "^5.9.0",
    "vitest": "^4.0.0"
  },
  "peerDependencies": {
    "openclaw": "*"
  }
}
```

### src/index.ts

```typescript
import type { Plugin, PluginContext } from 'openclaw/plugin-sdk';
import { MyExtensionChannel } from './channel.js';

export default {
  id: 'my-extension',
  version: '1.0.0',
  
  async initialize(context: PluginContext) {
    const channel = new MyExtensionChannel(context);
    context.channels.register('my-ext', channel);
    
    context.logger.info('My Extension loaded');
  }
} as Plugin;
```

### Instalar Plugin

```bash
# Copiar a extensions/
cp -r my-extension extensions/

# Install
pnpm install

# Rebuild
pnpm build
```

---

## 🧪 Testing

### Configuraciones de Test

```bash
# Unit tests (default)
pnpm test

# E2E tests
pnpm test:e2e

# Tests en vivo (requiere LIVE=1)
LIVE=1 pnpm test:live

# Con cobertura
pnpm test:coverage

# Watch mode
pnpm test:watch

# Tests específicos
pnpm test -- pattern-match

# Docker tests
pnpm test:docker:live-models
```

### Escribir Tests Robustos

```typescript
import { describe, it, expect, beforeEach, vi } from 'vitest';

describe('Feature X', () => {
  let service: Service;
  
  beforeEach(() => {
    // Setup
    service = new Service();
  });
  
  it('should handle happy path', async () => {
    const result = await service.doSomething();
    
    expect(result).toEqual({
      success: true,
      data: expect.any(Object)
    });
  });
  
  it('should handle errors', async () => {
    // Simular error
    vi.spyOn(service, 'method').mockRejectedValue(
      new Error('API Error')
    );
    
    await expect(service.doSomething()).rejects.toThrow('API Error');
  });
  
  it('should retry on failure', async () => {
    const spy = vi.fn();
    
    // Primera llamada falla
    spy.mockRejectedValueOnce(new Error('Fail'));
    
    // Segunda llamada éxito
    spy.mockResolvedValueOnce({ success: true });
    
    const result = await withRetry(() => spy());
    
    expect(result.success).toBe(true);
    expect(spy).toHaveBeenCalledTimes(2);
  });
});
```

---

## 📊 Performance & Profiling

### Monitoreo

```bash
# Ver memory usage
ps aux | grep node

# Monitor en tiempo real
node --expose-gc scripts/monitor.js
```

### Logging Detallado

```bash
# Verbose mode
OPENCLAW_DEBUG=1 pnpm dev

# Con timestamps
DEBUG=openclaw:* node script.js

# Salida a archivo
pnpm dev > debug.log 2>&1
```

---

## 🚀 Preparar para Publicación

### Pre-publicación

```bash
# Verificar version
cat package.json | grep version

# Build completo
pnpm build

# Tests completos
pnpm lint && pnpm test && pnpm test:e2e

# Generar changelog
# (Agregar entry a CHANGELOG.md)

# Verificar no hay cambios uncommitted
git status

# Commit de changelog
git add CHANGELOG.md
git commit -m "chore: release vX.Y.Z"
```

### Publicar a npm

```bash
# Tag release
git tag vX.Y.Z
git push origin vX.Y.Z

# Publicar (requiere acceso a npm)
npm publish --access public

# Verificar publicación
npm view openclaw@X.Y.Z
```

---

## 🐛 Debugging

### VS Code Launch Config

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "OpenClaw Dev",
      "program": "${workspaceFolder}/scripts/run-node.mjs",
      "args": ["gateway", "--verbose"],
      "console": "integratedTerminal",
      "internalConsoleOptions": "neverOpen"
    },
    {
      "type": "node",
      "request": "launch",
      "name": "Run Tests",
      "program": "${workspaceFolder}/node_modules/.bin/vitest",
      "console": "integratedTerminal"
    }
  ]
}
```

### Breakpoints

```typescript
// Usar debugger; en código
debugger;

// Breakpoint condicional en tests
it('test', () => {
  debugger; // Se pausará aquí si ejecutas con debugger
  expect(x).toBe(y);
});
```

### Inspeccionar Estado

```bash
# Node inspector
node --inspect scripts/run-node.mjs gateway

# Chrome devtools
# -> chrome://inspect
```

---

## 📚 Recursos Útiles

### Documentación Interna
- `docs/` - Documentación de usuario
- `CLAUDE.md` - Guía para Claude
- `AGENTS.md` - Guidelines para agentes
- `CONTRIBUTING.md` - Guía de contribución

### Repositorios Relacionados
- https://github.com/openclaw/openclaw - Main repo
- https://github.com/openclaw/nix-clawdbot - Nix packaging
- https://github.com/openclaw/openclaw.ai - Website

### Dependencias Importantes
- [@mariozechner/pi-agent-core](npm/@mariozechner/pi-agent-core) - Pi core
- [@whiskeysockets/baileys](npm/@whiskeysockets/baileys) - WhatsApp
- [grammy](npm/grammy) - Telegram
- [commander](npm/commander) - CLI

---

## ✅ Checklist para Contribuciones

- [ ] Código sigue estilo del proyecto (ran `pnpm lint:fix`)
- [ ] Tests escritos y pasando (`pnpm test`)
- [ ] Build exitoso (`pnpm build`)
- [ ] Changelog actualizado si user-facing
- [ ] Commits con mensajes claros
- [ ] No hay conflictos con `main`
- [ ] PR tiene descripción clara

---

Este documento es una guía viva. Se actualiza conforme evoluciona el proyecto.

