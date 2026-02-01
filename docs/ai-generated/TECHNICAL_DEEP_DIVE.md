# OpenClaw - Guía Técnica Profunda

## 📋 Índice
1. [Flujos de Datos](#flujos-de-datos)
2. [Patrones de Arquitectura](#patrones-de-arquitectura)
3. [Módulos Clave](#módulos-clave)
4. [APIs Críticas](#apis-críticas)
5. [Sistemas de Configuración](#sistemas-de-configuración)
6. [Manejo de Errores](#manejo-de-errores)
7. [Estrategia de Caching](#estrategia-de-caching)
8. [Protocolo de Comunicación](#protocolo-de-comunicación)

---

## 🔄 Flujos de Datos

### 1. Flujo de Mensaje Entrante

```
┌─────────────────────────────────────────────────────────────────┐
│ Canal Externo (WhatsApp/Telegram/Discord/Slack/Signal/etc)    │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                    ┌────────────▼──────────────┐
                    │  Adapter del Canal       │
                    │  (src/channels/*)        │
                    └────────────┬──────────────┘
                                 │
                    ┌────────────▼──────────────┐
                    │  Normalizar Mensaje      │
                    │  (metadata, attachments) │
                    └────────────┬──────────────┘
                                 │
                    ┌────────────▼──────────────┐
                    │  Gateway Registry        │
                    │  (channel-config.ts)     │
                    └────────────┬──────────────┘
                                 │
                    ┌────────────▼──────────────┐
                    │  Router                  │
                    │  (routing/targets.ts)    │
                    └────────────┬──────────────┘
                                 │
                    ┌────────────▼──────────────┐
                    │  Validación              │
                    │  - Allowlists            │
                    │  - Command gating        │
                    │  - Roles/Permisos        │
                    └────────────┬──────────────┘
                                 │
                    ┌────────────▼──────────────┐
                    │  Agent Dispatcher        │
                    │  (agents/index.ts)       │
                    └────────────┬──────────────┘
                                 │
                ┌────────────────┴────────────────┐
                │                                 │
    ┌───────────▼────────────┐      ┌────────────▼──────────────┐
    │  Load Session          │      │  Prepare Context         │
    │  (session-utils.ts)    │      │  (context.ts)            │
    └───────────┬────────────┘      └────────────┬──────────────┘
                │                                │
                └────────────────┬───────────────┘
                                 │
                    ┌────────────▼──────────────┐
                    │  Build System Prompt     │
                    │  (system-prompt.ts)      │
                    └────────────┬──────────────┘
                                 │
                    ┌────────────▼──────────────────────────────┐
                    │  Run Pi Embedded Agent                    │
                    │  (pi-embedded-runner.ts)                  │
                    │                                           │
                    │  1. Prepare tools                         │
                    │  2. Call LLM (with streaming)             │
                    │  3. Parse tool calls                      │
                    │  4. Execute tools                         │
                    │  5. Loop until done                       │
                    └────────────┬──────────────────────────────┘
                                 │
                    ┌────────────▼──────────────┐
                    │  Subscribe to Events     │
                    │  (pi-embedded-subscribe) │
                    │  - Streaming             │
                    │  - Tool execution        │
                    │  - Final answer          │
                    └────────────┬──────────────┘
                                 │
                    ┌────────────▼──────────────┐
                    │  Format Response         │
                    │  (reply formatting)      │
                    └────────────┬──────────────┘
                                 │
                ┌────────────────┴───────────────┐
                │                                │
    ┌───────────▼──────────────┐    ┌───────────▼──────────────┐
    │  Guardar en Sesión       │    │  Enviar a Canales       │
    │  (session persistence)   │    │  (channel adapters)     │
    └──────────────────────────┘    └───────────┬──────────────┘
                                                 │
                                   ┌─────────────▼────────────┐
                                   │  Canal Remoto           │
                                   │  (WhatsApp/Telegram...) │
                                   └─────────────────────────┘
```

### 2. Flujo de Ejecución de Herramientas

```
Tool Call detectado en respuesta LLM
           │
    ┌──────▼──────┐
    │ Tool ID     │
    │ Validation  │
    └──────┬──────┘
           │
    ┌──────▼──────────────────┐
    │ Tool Policy Check       │
    │ (tool-policy.ts)        │
    │ - Allowlist validation  │
    │ - Arg validation        │
    │ - Resource limits       │
    └──────┬──────────────────┘
           │
    ┌──────▼──────────────────┐
    │ Tool Type               │
    ├──────────────────────────┤
    │ 1. bash/exec (PTY)      │
    │ 2. browser/screenshot   │
    │ 3. memory/search        │
    │ 4. camera/image capture │
    │ 5. messaging/send       │
    │ 6. custom (SDK tools)   │
    └──────┬──────────────────┘
           │
    ┌──────▼──────────────────┐
    │ Execute in Sandbox      │
    │ - Env vars limited      │
    │ - Resources capped      │
    │ - Timeout enforcement   │
    └──────┬──────────────────┘
           │
    ┌──────▼──────────────────┐
    │ Capture Output          │
    │ - stdout/stderr         │
    │ - Exit code             │
    │ - Attachments           │
    └──────┬──────────────────┘
           │
    ┌──────▼──────────────────┐
    │ Format Tool Result      │
    │ (session-tool-result-   │
    │  guard.ts)              │
    └──────┬──────────────────┘
           │
    ┌──────▼──────────────────┐
    │ Return to Agent         │
    │ Trigger next loop       │
    └─────────────────────────┘
```

---

## 🏗️ Patrones de Arquitectura

### 1. Inyección de Dependencias

```typescript
// src/cli/deps.ts
interface Deps {
  config: Config;
  channels: ChannelRegistry;
  providers: ProviderRegistry;
  storage: Storage;
  logger: Logger;
}

// Factory pattern
function createDefaultDeps(): Deps {
  const config = loadConfig();
  const channels = initializeChannels(config);
  const providers = initializeProviders(config);
  // ...
  return { config, channels, providers, storage, logger };
}

// Usage
const deps = createDefaultDeps();
await gateway(deps);
```

**Ventajas:**
- Testable (mock dependencies)
- Desacoplado
- Configurable
- Reutilizable

---

### 2. Registry Pattern

```typescript
// src/channels/registry.ts
class ChannelRegistry {
  private channels: Map<string, Channel> = new Map();
  
  register(id: string, channel: Channel): void {
    this.channels.set(id, channel);
  }
  
  get(id: string): Channel | undefined {
    return this.channels.get(id);
  }
  
  getAll(): Channel[] {
    return Array.from(this.channels.values());
  }
}

// Usage
const registry = new ChannelRegistry();
registry.register('whatsapp', whatsappChannel);
registry.register('telegram', telegramChannel);
const wp = registry.get('whatsapp');
```

**Usado para:**
- Canales (`src/channels/registry.ts`)
- Modelos (`src/providers/model-catalog.ts`)
- Plugins (`src/gateway/server-plugins.ts`)
- Subagenttes (`src/agents/subagent-registry.ts`)

---

### 3. Strategy Pattern

```typescript
// src/agents/model-selection.ts
interface ModelSelectionStrategy {
  select(options: SelectOptions): Promise<Model>;
}

class RoundRobinStrategy implements ModelSelectionStrategy {
  async select(options: SelectOptions): Promise<Model> {
    // Round-robin entre modelos disponibles
  }
}

class PriorityStrategy implements ModelSelectionStrategy {
  async select(options: SelectOptions): Promise<Model> {
    // Selecciona por prioridad configurada
  }
}

// Usage
const strategy = new RoundRobinStrategy();
const model = await strategy.select(options);
```

---

### 4. Observer Pattern (Event-Driven)

```typescript
// src/gateway/server-chat.ts
class ChatManager {
  private listeners: Set<ChatListener> = new Set();
  
  subscribe(listener: ChatListener): void {
    this.listeners.add(listener);
  }
  
  private notifyListeners(event: ChatEvent): void {
    for (const listener of this.listeners) {
      listener.onChatEvent(event);
    }
  }
}

// Usage
chatManager.subscribe({
  onChatEvent: (event) => {
    if (event.type === 'message') {
      handleMessage(event);
    }
  }
});
```

---

### 5. Factory Pattern

```typescript
// src/channels/plugins/create-channel.ts
async function createChannelFactory(
  id: string,
  config: ChannelConfig
): Promise<Channel> {
  switch (config.type) {
    case 'whatsapp':
      return new WhatsAppChannel(config);
    case 'telegram':
      return new TelegramChannel(config);
    case 'slack':
      return new SlackChannel(config);
    // ... etc
    default:
      throw new Error(`Unknown channel: ${config.type}`);
  }
}
```

---

### 6. Middleware Pattern

```typescript
// src/gateway/server-chat.ts
type ChatMiddleware = (
  message: Message,
  next: () => Promise<void>
) => Promise<void>;

const middlewares: ChatMiddleware[] = [
  validateAllowlist,
  checkCommandGating,
  enrichMetadata,
  routeToAgent,
  formatResponse,
  sendToChannel
];

async function processMessage(message: Message) {
  let index = 0;
  const next = async () => {
    if (index < middlewares.length) {
      await middlewares[index++](message, next);
    }
  };
  await next();
}
```

---

## 🔑 Módulos Clave

### 1. src/agents/pi-embedded-runner.ts

**Responsabilidad**: Ejecutor principal del agente Pi

```typescript
interface EmbeddedAgentOptions {
  sessionId: string;
  model: Model;
  systemPrompt: string;
  userMessage: string;
  tools: Tool[];
  context?: ConversationContext;
  maxIterations?: number;
  abortSignal?: AbortSignal;
}

async function runEmbeddedPiAgent(
  options: EmbeddedAgentOptions
): Promise<AgentResult> {
  // 1. Preparar contexto
  const context = buildContext(options);
  
  // 2. Preparar tools
  const tools = prepareTools(options.tools);
  
  // 3. Loop principal
  while (!done && iterations < maxIterations) {
    // 4. Llamar LLM
    const response = await callModel(model, {
      messages: context.messages,
      tools,
      systemPrompt: options.systemPrompt,
      maxTokens: calculateMaxTokens(context)
    });
    
    // 5. Procesar respuesta
    if (response.type === 'tool_use') {
      const result = await executeTool(response.tool);
      context.messages.push({
        role: 'user',
        content: result
      });
    } else if (response.type === 'text') {
      return response;
    }
    
    iterations++;
  }
  
  return context.finalMessage;
}
```

**Estado** (Context):
- `sessionId`: ID de la sesión
- `messages`: Historial de conversación
- `toolResults`: Resultados de herramientas
- `metadata`: Metadata de contexto

---

### 2. src/gateway/server-chat.ts

**Responsabilidad**: Gestión centralizada de chats

```typescript
class ChatManager {
  async handleIncomingMessage(
    message: IncomingMessage
  ): Promise<void> {
    // 1. Validar
    await validateMessage(message);
    
    // 2. Enriquecer
    const enrichedMessage = await enrichMetadata(message);
    
    // 3. Obtener sesión
    const session = await getOrCreateSession(
      enrichedMessage.sender,
      enrichedMessage.channel
    );
    
    // 4. Ejecutar agente
    const response = await this.agentDispatcher.dispatch({
      session,
      userMessage: enrichedMessage.text,
      attachments: enrichedMessage.attachments
    });
    
    // 5. Guardar respuesta
    await session.save(response);
    
    // 6. Enviar a canales
    await this.channelManager.send(response);
  }
}
```

---

### 3. src/routing/targets.ts

**Responsabilidad**: Enrutamiento inteligente de mensajes

```typescript
interface RoutingTarget {
  channelId: string;
  recipientId: string;
  senderId?: string;
  sessionId?: string;
}

function resolveRoutingTargets(
  message: Message,
  config: RoutingConfig
): RoutingTarget[] {
  // 1. Determinar canal principal
  const primaryTarget = {
    channelId: message.channel,
    recipientId: message.sender
  };
  
  // 2. Determinar canales secundarios
  const secondaryTargets = config.routingRules
    .filter(rule => rule.matches(message))
    .map(rule => rule.getTarget());
  
  // 3. Deduplicar
  return deduplicateTargets([
    primaryTarget,
    ...secondaryTargets
  ]);
}
```

---

### 4. src/agents/auth-profiles/auth-profiles.ts

**Responsabilidad**: Gestión inteligente de perfiles de autenticación

```typescript
interface AuthProfile {
  providerId: string;
  apiKey: string;
  baseUrl?: string;
  priority?: number;
  cooldown?: number;
  lastUsed?: Date;
  lastFailed?: Date;
}

class AuthProfileManager {
  async selectProfile(
    providerId: string,
    options?: SelectOptions
  ): Promise<AuthProfile> {
    const profiles = this.getProfilesForProvider(providerId);
    
    // 1. Filtrar disponibles
    const available = profiles.filter(p => 
      !this.isCoolingDown(p) && !this.isFailing(p)
    );
    
    if (available.length === 0) {
      throw new Error('No available profiles');
    }
    
    // 2. Ordenar por estrategia
    const sorted = this.sortByStrategy(available, options);
    
    // 3. Retornar mejor opción
    return sorted[0];
  }
  
  async markFailure(profile: AuthProfile): Promise<void> {
    profile.lastFailed = new Date();
    profile.cooldown = exponentialBackoff(profile.failures);
    await this.save(profile);
  }
}
```

---

### 5. src/memory/index.ts

**Responsabilidad**: Gestión de memoria persistente

```typescript
interface MemoryStore {
  // Búsqueda semántica
  search(query: string, limit?: number): Promise<MemoryEntry[]>;
  
  // Guardar en vectores
  add(content: string, metadata?: Record<string, any>): Promise<string>;
  
  // Recuperar por ID
  get(id: string): Promise<MemoryEntry | null>;
  
  // Actualizar
  update(id: string, content: string): Promise<void>;
  
  // Eliminar
  delete(id: string): Promise<void>;
}

class HybridMemoryStore implements MemoryStore {
  constructor(
    private vectorDb: VectorDB,  // LanceDB
    private metaDb: Database     // SQLite
  ) {}
  
  async search(query: string, limit?: number) {
    // 1. Embeddings query
    const embedding = await this.getEmbedding(query);
    
    // 2. Búsqueda en vectores
    const vectors = await this.vectorDb.search(
      embedding,
      limit
    );
    
    // 3. Enriquecer con metadata
    const results = await Promise.all(
      vectors.map(v => this.metaDb.get(v.id))
    );
    
    return results.filter(Boolean);
  }
}
```

---

### 6. src/channels/registry.ts

**Responsabilidad**: Centralizar acceso a canales

```typescript
class ChannelRegistry {
  private channels: Map<string, Channel> = new Map();
  
  async initialize(config: ChannelsConfig): Promise<void> {
    for (const [id, channelConfig] of Object.entries(config)) {
      try {
        const channel = await createChannel(id, channelConfig);
        this.register(id, channel);
      } catch (error) {
        logger.error(`Failed to initialize channel ${id}`, error);
      }
    }
  }
  
  async send(message: Message, target: RoutingTarget): Promise<void> {
    const channel = this.get(target.channelId);
    if (!channel) {
      throw new Error(`Channel not found: ${target.channelId}`);
    }
    
    await channel.send(message, {
      recipientId: target.recipientId,
      senderId: target.senderId
    });
  }
}
```

---

## 🔌 APIs Críticas

### 1. Plugin API (Plugin SDK)

```typescript
// dist/plugin-sdk/index.ts

export interface PluginContext {
  gateway: GatewayClient;
  config: Config;
  logger: Logger;
  channels: ChannelRegistry;
  models: ModelRegistry;
}

export interface Plugin {
  id: string;
  version: string;
  
  initialize(context: PluginContext): Promise<void>;
  
  getChannels?(): Channel[];
  getModels?(): ModelProvider[];
  getTools?(): Tool[];
  getHooks?(): Hook[];
  
  shutdown?(): Promise<void>;
}

// Usage en extensión
export default {
  id: 'my-extension',
  version: '1.0.0',
  
  async initialize(context: PluginContext) {
    const channel = new MyCustomChannel(context.config);
    context.channels.register('my-channel', channel);
  }
} as Plugin;
```

---

### 2. Channel Interface

```typescript
export interface Channel {
  id: string;
  name: string;
  
  // Lifecycle
  connect(): Promise<void>;
  disconnect(): Promise<void>;
  isConnected(): boolean;
  
  // Sending
  send(message: Message, target: SendTarget): Promise<void>;
  
  // Receiving
  onMessage(listener: (msg: Message) => void): void;
  
  // Metadata
  getIdentity(): Identity;
  getCapabilities(): ChannelCapabilities;
}
```

---

### 3. Tool Interface

```typescript
export interface Tool {
  id: string;
  name: string;
  description: string;
  inputSchema: JSONSchema;
  
  execute(
    input: unknown,
    context: ToolExecutionContext
  ): Promise<ToolResult>;
}

export interface ToolResult {
  type: 'text' | 'image' | 'file' | 'error';
  content: unknown;
  metadata?: Record<string, any>;
}
```

---

## ⚙️ Sistemas de Configuración

### 1. Config Loading

```typescript
// src/config/config.ts

interface Config {
  gateway: GatewayConfig;
  channels: ChannelsConfig;
  providers: ProvidersConfig;
  memory: MemoryConfig;
  security: SecurityConfig;
}

async function loadConfig(): Promise<Config> {
  // 1. Defaults
  const defaults = DEFAULT_CONFIG;
  
  // 2. Load from file ~/.openclaw/config.yaml
  const fileConfig = await loadYamlConfig(
    path.join(os.homedir(), '.openclaw', 'config.yaml')
  );
  
  // 3. Environment variables override
  const envConfig = loadEnvConfig();
  
  // 4. Merge with validation
  return validateConfig(
    deepMerge(defaults, fileConfig, envConfig)
  );
}
```

### 2. Validación con Zod

```typescript
// src/config/schemas.ts

const ChannelConfigSchema = z.object({
  enabled: z.boolean().default(true),
  type: z.enum(['whatsapp', 'telegram', 'slack', /* ... */]),
  credentials: z.record(z.string()),
  settings: z.record(z.unknown()).optional(),
});

const ConfigSchema = z.object({
  gateway: GatewayConfigSchema,
  channels: z.record(ChannelConfigSchema),
  providers: z.record(ProviderConfigSchema),
});

type Config = z.infer<typeof ConfigSchema>;
```

---

## ⚠️ Manejo de Errores

### 1. Hierarchy de Errores

```typescript
// src/infra/errors.ts

abstract class OpenClawError extends Error {
  constructor(
    message: string,
    public code: string,
    public severity: 'error' | 'warn' | 'info',
    public context?: Record<string, any>
  ) {
    super(message);
  }
}

class ChannelError extends OpenClawError {
  constructor(message: string, context?: any) {
    super(message, 'CHANNEL_ERROR', 'error', context);
  }
}

class ModelAuthError extends OpenClawError {
  constructor(message: string, context?: any) {
    super(message, 'MODEL_AUTH_ERROR', 'error', context);
  }
}

class ConfigError extends OpenClawError {
  constructor(message: string, context?: any) {
    super(message, 'CONFIG_ERROR', 'error', context);
  }
}
```

### 2. Error Recovery

```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  options: RetryOptions = {}
): Promise<T> {
  const {
    maxAttempts = 3,
    delay = 1000,
    exponential = true,
    onRetry = () => {}
  } = options;
  
  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxAttempts - 1) throw error;
      
      onRetry(attempt, error);
      
      const waitTime = exponential
        ? delay * Math.pow(2, attempt)
        : delay;
      
      await new Promise(resolve => setTimeout(resolve, waitTime));
    }
  }
}
```

---

## 💾 Estrategia de Caching

### 1. Multi-Level Caching

```typescript
// src/agents/cache-trace.ts

interface CacheEntry<T> {
  value: T;
  expiresAt: number;
  hits: number;
  lastAccessed: number;
}

class MultiLevelCache<T> {
  private memory: Map<string, CacheEntry<T>> = new Map();
  private disk: DiskCache<T>;
  private remote: RemoteCache<T>;
  
  async get(key: string): Promise<T | null> {
    // 1. Memory (fast)
    if (this.memory.has(key)) {
      return this.memory.get(key)!.value;
    }
    
    // 2. Disk (medium)
    const diskValue = await this.disk.get(key);
    if (diskValue) {
      this.memory.set(key, {
        value: diskValue,
        expiresAt: Date.now() + 300000,
        hits: 0,
        lastAccessed: Date.now()
      });
      return diskValue;
    }
    
    // 3. Remote (slow)
    const remoteValue = await this.remote.get(key);
    if (remoteValue) {
      await this.disk.set(key, remoteValue);
      return remoteValue;
    }
    
    return null;
  }
}
```

### 2. Cache Invalidation

```typescript
// Invalidar cache en cambios de config
config.onChanged(() => {
  cache.invalidate('models:*');
  cache.invalidate('config:*');
});

// TTL automático
const modelCache = new TTLCache<Model>(
  5 * 60 * 1000 // 5 minutos
);

// Invalidación manual
await cache.delete('auth-profiles:*');
```

---

## 🔗 Protocolo de Comunicación

### 1. WebSocket Protocol

```typescript
// src/gateway/protocol/types.ts

type ServerMessage =
  | { type: 'chat.event'; payload: ChatEvent }
  | { type: 'agent.update'; payload: AgentUpdate }
  | { type: 'tool.result'; payload: ToolResult }
  | { type: 'session.updated'; payload: SessionUpdate }
  | { type: 'error'; payload: ErrorPayload }
  | { type: 'auth.required'; payload: AuthChallenge };

type ClientMessage =
  | { type: 'chat.send'; payload: SendPayload }
  | { type: 'agent.command'; payload: CommandPayload }
  | { type: 'config.patch'; payload: ConfigPatchPayload }
  | { type: 'session.select'; payload: { sessionId: string } }
  | { type: 'ping' };

// Message flow
// Client -> Server
wss.on('message', (msg: ClientMessage) => {
  switch (msg.type) {
    case 'chat.send':
      await chatManager.handleIncoming(msg.payload);
      break;
    // ...
  }
});

// Server -> Client
function broadcast(message: ServerMessage) {
  for (const client of wss.clients) {
    client.send(JSON.stringify(message));
  }
}
```

### 2. HTTP API

```typescript
// src/gateway/server-http.ts

app.post('/api/chat/send', authenticate, async (req, res) => {
  const { message, target, attachments } = req.body;
  
  try {
    const result = await chatManager.send({
      text: message,
      target,
      attachments
    });
    
    res.json({ success: true, result });
  } catch (error) {
    res.status(500).json({ 
      success: false,
      error: formatError(error)
    });
  }
});

app.get('/api/sessions', authenticate, async (req, res) => {
  const sessions = await sessionManager.listAll();
  res.json(sessions);
});

app.get('/api/models', authenticate, async (req, res) => {
  const models = await modelCatalog.getAvailable();
  res.json(models);
});
```

---

## 🎯 Patrones de Composición

### 1. Composición de Herramientas

```typescript
// Herramientas incluidas en toda sesión
const baseTool: Tool[] = [
  // Bash execution
  new BashTool(),
  
  // Browser automation
  new ScreenshotTool(),
  new ClickTool(),
  new TypeTool(),
  
  // Memory operations
  new MemorySearchTool(),
  new MemoryAddTool(),
  
  // Messaging
  new SendMessageTool(),
  new GetContactsTool(),
];

// Herramientas por sesión/usuario
const sessionTools = baseTool.concat([
  ...loadUserDefinedTools(user.id),
  ...loadPluginTools(plugins),
  ...loadSDKTools(sdkDefinitions)
]);
```

---

## 📊 Métricas y Observabilidad

### 1. Structured Logging

```typescript
// src/logging.ts

interface LogEntry {
  timestamp: string;
  level: 'debug' | 'info' | 'warn' | 'error';
  module: string;
  message: string;
  context?: Record<string, any>;
  traceId?: string;
  userId?: string;
  sessionId?: string;
  duration?: number;
}

const logger = new Logger({
  minLevel: LogLevel.Info,
  transportOptions: {
    fs: {
      file: '~/.openclaw/logs/openclaw.log'
    }
  }
});

// Usage
logger.info('Message received', {
  senderId: msg.sender,
  channelId: msg.channel,
  length: msg.text.length
});
```

---

Este documento proporciona una visión profunda de cómo OpenClaw está arquitecturado internamente.

