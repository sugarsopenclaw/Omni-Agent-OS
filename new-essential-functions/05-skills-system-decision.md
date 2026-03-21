# 05 - Skills 体系设计 架构决策

**技术点**: Skills 体系设计

**分析日期**: 2026-03-21

---

## 1. 已有实现

### 1.1 IronClaw (Rust) — SKILL.md 信任模型

**位置**: `reference-source-code/personal-general-ai/ironclaw/src/skills/mod.rs`

**核心设计**:

```rust
/// 技能信任等级
pub enum SkillTrust {
    Installed = 0,  // 注册表/外部技能：只读工具
    Trusted = 1,    // 用户放置技能：完整工具访问
}

/// 激活条件
pub struct ActivationCriteria {
    pub keywords: Vec<String>,      // 关键词触发 (最多 20 个)
    pub exclude_keywords: Vec<String>, // 排除关键词
    pub patterns: Vec<String>,      // 正则模式 (最多 5 个)
    pub tags: Vec<String>,          // 标签 (最多 10 个)
    pub max_context_tokens: usize,  // 最大上下文 token
}

/// 技能清单
pub struct SkillManifest {
    pub name: String,
    pub version: String,
    pub description: String,
    pub activation: ActivationCriteria,
}
```

**SKILL.md 格式**:
```yaml
---
name: web-search
version: 1.0.0
activation:
  keywords: ["search", "google"]
  tags: ["web", "search"]
---

# Web Search

You can search the web...
```

**特点**: YAML frontmatter + Markdown、信任等级、关键词激活

---

### 1.2 OpenClaw (TypeScript) — 工作区技能

**位置**: `reference-source-code/personal-general-ai/openclaw/src/agents/skills.ts`

**核心设计**:

```typescript
interface SkillEntry {
  path: string;
  metadata: OpenClawSkillMetadata;
  content: string;
  enabled: boolean;
}

interface SkillSnapshot {
  entries: SkillEntry[];
  combinedPrompt: string;
  totalTokens: number;
}
```

**特点**: 工作区管理、技能快照、Token 预算

---

### 1.3 IronClaw 技能选择器

**位置**: `reference-source-code/personal-general-ai/ironclaw/src/skills/selector.rs`

```rust
pub struct SkillScore {
    pub skill: LoadedSkill,
    pub score: f64,
    pub matched_keywords: Vec<String>,
}

// 评分算法
// 关键词匹配: +0.3
// 正则匹配: +0.5
// 标签匹配: +0.2
// 排除关键词: 直接排除
```

**特点**: 多维度评分、排除机制

---

### 1.4 IronClaw 工具权限衰减

**位置**: `reference-source-code/personal-general-ai/ironclaw/src/skills/attenuation.rs`

```rust
pub fn attenuate_tools(
    available_tools: &[Arc<dyn Tool>],
    active_skills: &[LoadedSkill],
) -> AttenuationResult {
    // 最低信任等级决定工具上限
    let min_trust = active_skills.iter().map(|s| s.trust).min();
    
    match min_trust {
        SkillTrust::Trusted => // 全部允许
        SkillTrust::Installed => // 只允许只读工具
    }
}
```

**特点**: 最低信任决定权限、防止权限提升

---

## 2. 值得直接复用

| 项目 | 特性 | 推荐指数 |
|------|------|----------|
| IronClaw | SKILL.md 格式 | ⭐⭐⭐⭐⭐ |
| IronClaw | 信任模型 | ⭐⭐⭐⭐⭐ |
| IronClaw | 评分算法 | ⭐⭐⭐⭐ |
| OpenClaw | 技能快照 | ⭐⭐⭐⭐ |

---

## 3. 需要垂直特化

### 3.1 领域特定技能库

- 医疗: diagnosis-assistant, prescription-check
- 法律: case-law-search, contract-review
- 金融: market-data, risk-assessment
- 教育: quiz-generator, explanation-tutor

### 3.2 技能依赖管理

```yaml
---
name: medical-diagnosis
requires:
  skills: ["symptom-checker", "drug-database"]
  tools: ["read_file", "web_search"]
---
```

---

## 4. 推荐接口

```rust
struct Skill {
    manifest: SkillManifest,
    content: String,
    trust: SkillTrust,
}

struct SkillRegistry {
    skills: HashMap<String, Skill>,
}

impl SkillRegistry {
    fn load(&mut self, path: &Path) -> Result<Skill>;
    fn select(&self, query: &str) -> Vec<SkillScore>;
    fn get_prompt(&self, skills: &[Skill]) -> String;
}
```

---

## 5. MVP 方案

```python
class SimpleSkill:
    def __init__(self, path: Path):
        self.manifest = yaml.safe_load(path.read_text().split('---')[1])
        self.content = path.read_text().split('---')[2]
    
    def matches(self, query: str) -> bool:
        return any(kw in query for kw in self.manifest['keywords'])

class SkillRegistry:
    def __init__(self):
        self.skills: list[SimpleSkill] = []
    
    def load(self, dir: Path):
        for path in dir.glob('*/SKILL.md'):
            self.skills.append(SimpleSkill(path))
    
    def select(self, query: str) -> list[SimpleSkill]:
        return [s for s in self.skills if s.matches(query)]
```

---

## 6. 扩展路线

1. 信任模型
2. 评分算法
3. 工具权限衰减
4. 技能依赖

---

*文档版本：1.0*
*最后更新：2026-03-21*
