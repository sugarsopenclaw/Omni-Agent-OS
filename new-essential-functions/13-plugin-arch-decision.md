# 13 - Plugin Architecture 架构决策

**技术点**: Plugin Architecture

**分析日期**: 2026-03-21

---

## 1. 已有实现

### 1.1 IronClaw (Rust) — WASM Sandbox

**位置**: `reference-source-code/personal-general-ai/ironclaw/src/tools/wasm/`

**核心设计**:

```rust
/// WASM 运行时配置
pub struct WasmRuntimeConfig {
    pub max_memory_pages: u32,      // 128MB default
    pub max_fuel: u64,               // CPU fuel limit
    pub enable_wasi: bool,
    pub network_allowlist: Vec<String>,
}

/// WASM 工具运行时
pub struct WasmToolRuntime {
    engine: Engine,
    module_cache: Arc<RwLock<HashMap<String, Module>>>,
    config: WasmRuntimeConfig,
}

impl WasmToolRuntime {
    pub async fn prepare(&self, name: &str, wasm_bytes: &[u8], capabilities: Option<Capabilities>) 
        -> Result<PreparedModule> 
    {
        // 1. 编译模块 (缓存)
        let module = self.get_or_compile_module(name, wasm_bytes).await?;
        
        // 2. 创建 WASI 上下文
        let wasi = WasiCtxBuilder::new()
            .preopened_dir(allowed_dir, "/workspace", perms)?
            .build();
        
        // 3. 创建 store (fuel limit)
        let mut store = Store::new(&self.engine, wasi);
        store.add_fuel(self.config.max_fuel)?;
        
        // 4. 实例化
        let instance = self.linker.instantiate(&mut store, &module)?;
        
        Ok(PreparedModule { instance, store, capabilities })
    }
}

/// 安全约束
/// | 威胁 | 缓解措施 |
/// | CPU 耗尽 | Fuel metering |
/// | 内存耗尽 | ResourceLimiter (10MB) |
/// | 无限循环 | Epoch interruption + timeout |
/// | 文件系统访问 | 仅 host workspace_read |
/// | 网络访问 | Allowlist only |
/// | 凭证暴露 | Host boundary injection |
```

**特点**:
- wasmtime 沙箱
- Fuel metering (CPU 限制)
- 能力系统 (Capabilities)
- 凭证注入 (边界安全)

---

### 1.2 OpenClaw (TypeScript) — Plugin SDK

**位置**: `reference-source-code/personal-general-ai/openclaw/src/gateway/server-plugins.ts`

**核心设计**:

```typescript
/// Plugin 运行时
export interface PluginRuntime {
  id: string;
  version: string;
  entry: string;
  config: PluginConfig;
}

/// Plugin 加载
export async function loadOpenClawPlugins(
  config: OpenClawConfig,
): Promise<PluginRuntime[]> {
  const plugins: PluginRuntime[] = [];
  
  for (const [id, entry] of Object.entries(config.plugins?.entries || {})) {
    if (!entry.enabled) continue;
    
    // 加载 Plugin
    const runtime = await loadPlugin(id, entry);
    plugins.push(runtime);
  }
  
  return plugins;
}

/// Plugin 子 Agent 策略
interface PluginSubagentOverridePolicy {
  allowModelOverride: boolean;
  allowAnyModel: boolean;
  hasConfiguredAllowlist: boolean;
  allowedModels: Set<string>;
}

/// 授权回退模型覆盖
function authorizeFallbackModelOverride(params: {
  pluginId?: string;
  provider?: string;
  model?: string;
}): { allowed: true } | { allowed: false; reason: string } {
  const policy = pluginSubagentPolicyState.policies[params.pluginId];
  
  if (!policy?.allowModelOverride) {
    return { allowed: false, reason: "Plugin not trusted for model override" };
  }
  
  if (policy.allowAnyModel) {
    return { allowed: true };
  }
  
  const requestedModelRef = resolveRequestedFallbackModelRef(params);
  if (policy.allowedModels.has(requestedModelRef)) {
    return { allowed: true };
  }
  
  return { allowed: false, reason: "Model not in allowlist" };
}
```

**特点**:
- Node.js Plugin
- 模型覆盖授权
- 允许列表

---

### 1.3 MCP (Model Context Protocol)

**通用模式**:

```typescript
/// MCP 客户端
interface MCPClient {
  connect(): Promise<void>;
  listTools(): Promise<ToolDefinition[]>;
  callTool(name: string, params: any): Promise<any>;
}

/// MCP 工具包装器
class MCPToolWrapper implements Tool {
  constructor(
    private serverName: string,
    private client: MCPClient,
    private toolDef: ToolDefinition,
  ) {}
  
  async execute(params: any): Promise<ToolResult> {
    return this.client.callTool(this.toolDef.name, params);
  }
}
```

---

## 2. 值得直接复用

### 2.1 IronClaw WASM Sandbox ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- 完整沙箱
- Fuel metering
- 能力系统

**复用方式**:
```rust
let runtime = WasmToolRuntime::new(config)?;
let prepared = runtime.prepare("tool", wasm_bytes, capabilities).await?;
let tool = WasmToolWrapper::new(runtime, prepared, capabilities);
```

---

### 2.2 OpenClaw Plugin SDK ⭐⭐⭐⭐

**推荐指数**: 4/5

**理由**:
- 运行时加载
- 授权策略

---

### 2.3 MCP Protocol ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- 标准化
- 生态丰富

---

## 3. 需要垂直特化

### 3.1 领域特定插件

- 医疗: HL7/FHIR 连接器
- 法律: 法条数据库
- 金融: 交易接口

### 3.2 合规审计

- 插件行为审计
- 数据访问日志
- 权限变更记录

---

## 4. 推荐接口

```rust
/// 插件管理器
pub trait PluginManager: Send + Sync {
    async fn load(&self, id: &str, path: &Path) -> Result<Plugin>;
    async fn unload(&self, id: &str) -> Result<()>;
    fn list(&self) -> Vec<Plugin>;
}

/// 插件
pub trait Plugin: Send + Sync {
    fn id(&self) -> &str;
    fn version(&self) -> &str;
    async fn initialize(&mut self) -> Result<()>;
    async fn shutdown(&mut self) -> Result<()>;
}

/// 安全沙箱
pub trait Sandbox: Send + Sync {
    async fn execute(&self, code: &[u8], params: Value) -> Result<Value>;
    fn set_resource_limits(&mut self, limits: ResourceLimits);
}
```

---

## 5. MVP 方案

```python
class SimplePluginManager:
    def __init__(self):
        self.plugins: dict[str, Plugin] = {}
    
    def load(self, plugin_id: str, module_path: Path):
        # 动态加载 Python 模块
        spec = importlib.util.spec_from_file_location(plugin_id, module_path)
        module = importlib.util.module_from_spec(spec)
        spec.loader.exec_module(module)
        
        self.plugins[plugin_id] = module
    
    def execute(self, plugin_id: str, function: str, params: dict):
        plugin = self.plugins.get(plugin_id)
        if not plugin:
            raise ValueError(f"Plugin {plugin_id} not found")
        
        func = getattr(plugin, function)
        return func(**params)
```

---

## 6. 扩展路线

1. WASM 沙箱
2. MCP 集成
3. 能力系统
4. 审计日志

---

*文档版本：1.0*
*最后更新：2026-03-21*
