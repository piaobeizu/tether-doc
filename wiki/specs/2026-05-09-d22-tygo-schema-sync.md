# D-22: Tygo 单向 schema 同步（Go → TypeScript）

**Date:** 2026-05-09
**Author:** wxk + Claude
**Status:** Proposed (in-flight，等用户拍板新方向后正式纳入主 spec)
**Companion docs:** `2026-04-26-tether-go-quic-design.md` 主 spec，
项目记忆 `project_tether_simplification_consideration.md`（in-flight pivot）

---

## 0. Why this exists

PoC step7（`/poc/go-quic-wt/step7_singlebinary.go`）落地过程中**同一类 bug 被独立踩了 2 次**：

1. `/cert-hash` 服务端返回 `aa:bb:cc:...` 带冒号格式，浏览器 JS 期望
   64 字符无分隔符 hex → 测试失败。
2. WT bidi stream 服务端 `Write([]byte("echo: " + buf))`，浏览器期望
   字节级纯回显 → 测试失败。

两个 bug 共同特征：**Go 端跟 TS 端各自对 wire format 有自己的理解，
没有共享 schema 强制对齐**。类型系统对这一类漂移**部分**有效——能保
证 field 名/type/optional 一致；但 string 内部的 format convention
（hash 格式、prefix 习惯）类型层抓不到。

D-22 是这个问题的 95% 解：用 **tygo** 把 Go struct 单向生成 TS interface，
共享 wire schema；剩下 5% 用 godoc + contract test 补。

## 1. 决策

**采纳 tygo 单向 schema 同步 (Go → TypeScript)**。具体：

- Wire 类型 single source of truth = `internal/wire/*.go`
- `tygo generate` 把它转成 `web/src/lib/wire.gen.ts`（或 PoC 阶段 `step3_browser/wire-types.gen.ts`）
- 前端 `import type { ... } from '../lib/wire.gen'`
- Build pipeline 把 `tygo generate` 放在 `vite build` 之前

**不采用** protobuf — 留作 v1.0+ 升级候选（理由见 §3）。

## 2. 落地清单

### 2.1 工具链

```bash
# 一次性安装
go install github.com/gzuidhof/tygo@latest
```

放在 README "Build prerequisites" 段，跟 Go / Node / pnpm 并列。

### 2.2 项目结构

```
tether/
├── internal/
│   ├── wire/                # ⚠ 仅含跨 daemon ↔ browser 的类型；被 tygo 生成
│   │   ├── types.go        # 跨前后端的 struct（Envelope / FencedBlock / ...）
│   │   ├── format.go       # type alias for format conventions (HashHex64, SessionID, ...)
│   │   └── doc.go          # package-level godoc
│   └── agent/               # daemon 内部翻译边界，**不**被 tygo 生成
│       └── event.go        # Event / EventKind / ToolUseEvent — provider 中性 normalize 形态
│                            # daemon 在发前端前把 Event 翻译成 wire.Envelope
├── tygo.yaml               # tygo 生成配置
├── web/
│   └── src/
│       └── lib/
│           └── wire.gen.ts # ⚠ AUTO-GENERATED — 不要手改
└── Makefile / scripts
```

### 2.3 tygo.yaml 模板

```yaml
packages:
  - path: github.com/piaobeizu/tether/internal/wire
    output_path: web/src/lib/wire.gen.ts
    type_mappings:
      time.Time: string         # ISO 8601 format on the wire
      json.RawMessage: unknown  # opaque payload
    frontmatter: |
      // ⚠ AUTO-GENERATED FROM internal/wire/*.go BY tygo. DO NOT EDIT.
      // To change shape: edit Go source then `make codegen`.
```

### 2.4 Build pipeline

```makefile
# Makefile
.PHONY: codegen build
codegen:
	tygo generate

web-build: codegen
	cd web && pnpm build

build: web-build
	go build -o bin/tether ./cmd/tether
```

`go generate ./...` 也可作为 `tygo generate` 的 alias，via:

```go
//go:generate tygo generate
package wire
```

### 2.5 Format conventions（类型系统抓不到的部分）

类型相同但**约定不同**的 string，用 type alias + godoc 强约束：

```go
// internal/wire/format.go
package wire

// HashHex64 is 64 lowercase hex characters, no separators.
//
// Browser parser: /^[0-9a-f]{64}$/
// Verified by contract test: web/test/wire-contract.spec.ts
type HashHex64 = string

// SessionID is a UUIDv4 in canonical lowercase form.
//
// Example: "b9c5e142-b0ad-4ba0-b8c7-e738761b9bc0"
type SessionID = string

// EnvelopeKind is one of: "control" | "events" | "agent-bytes"
//                       | "catch-up" | "datagram"
type EnvelopeKind = string
```

tygo 把 `type HashHex64 = string` 翻译成 TS `export type HashHex64 = string`
（含 godoc 注释作 JSDoc），人类可读、IDE 提示有用，但**没有 runtime 强制**。

强制留给 §2.6 contract test。

### 2.6 Wire-contract integration tests

`web/test/wire-contract.spec.ts` 里**直接打 server endpoint** 验证 format
约定。CI 跑这些。任何 format 漂移立即红。

```typescript
// web/test/wire-contract.spec.ts
import { describe, test, expect } from 'vitest'

describe('wire format contracts', () => {
  test('GET /cert-hash returns HashHex64 (64 lowercase hex)', async () => {
    const r = await fetch('http://127.0.0.1:8082/cert-hash')
    const text = (await r.text()).trim()
    expect(text).toMatch(/^[0-9a-f]{64}$/)
  })

  test('WT bidi stream pure echoes (no prefix, no framing)', async () => {
    const session = new WebTransport('https://127.0.0.1:4433/wt', {
      serverCertificateHashes: [{ algorithm: 'sha-256', value: ... }]
    })
    await session.ready
    const stream = await session.createBidirectionalStream()
    const writer = stream.writable.getWriter()
    const sent = new TextEncoder().encode('hello-from-browser')
    await writer.write(sent)
    await writer.close()
    const reader = stream.readable.getReader()
    const { value } = await reader.read()
    expect(new TextDecoder().decode(value)).toBe('hello-from-browser') // EXACT
  })
})
```

CI step 5 里跑这些（前面 4 步：`tygo generate`、`go build`、起 server、运行 vitest）。

### 2.7 PoC dogfood

step7 已经创建了示例：
- `poc/go-quic-wt/wire/types.go` — `HashHex64` 类型 + `EchoTestPayload`
- `poc/go-quic-wt/tygo.yaml` — 配置
- `poc/go-quic-wt/step3_browser/wire-types.gen.ts` — 生成产物（已 commit）

future 重 build step7 时，`/cert-hash` handler 应该 return type 标注为
`wire.HashHex64`。即便文档化收益（不是 runtime 强制），看代码的人立
刻知道格式约束。

## 3. 为啥不直接 protobuf

| 维度 | tygo（v0.1） | protobuf（v1.0+ 候选） |
|---|---|---|
| 工具链复杂度 | 1 个 binary，10 行 yaml | buf + protoc plugins + .proto syntax |
| Onboarding | 半天 | 1-2 周 |
| 单 Go binary 友好 | ✅ go install + 1 yaml | ❌ 需要 buf 安装 + 跨工具链协调 |
| 调试时 wire payload | 浏览器 DevTools 直接 JSON 可读 | binary 默认，需 plugin 才人读 |
| 双向 schema 强制 | 弱（编译期可见） | 强（`buf breaking` 自动） |
| 老客户端兼容性 | 弱 | 强（field number 规则） |
| Wire 序列化代码 | 手写 `JSON.parse` 一行 | 自动生成 |
| tether v0.1 实际需求 | ✅ 够用 | 过度工程 |

**升级触发条件**（满足其一就该考虑切 protobuf）：
- 多用户多设备（D-04 解锁）需要老客户端兼容
- JSON 序列化成性能瓶颈（unlikely 但有可能）
- 接 gRPC 给第三方 SDK 用

切的时候：把 `internal/wire/types.go` 翻译成 `wire.proto`，前端 import
path 改一下。Schema 内容大致 1:1 对应。**不会被 D-22 决策锁死**。

## 4. 跟其他 D-XX 的关系

- **D-14（Wire 协议 envelope-based）**：D-22 是 D-14 的工程实现机制。
  Envelope schema 在 `internal/wire/envelope.go`，前后端共享一份生成 TS。
- **D-18（v0.2 forward-compat）**：D-18 列的 9 项 seam（`workspaceRoot`、
  `/blob/*` 路由、`catchup.state-snapshot` kind 等）都是 wire 字段。它们
  的 JSON 名字现在变成"内嵌在 Go struct 里 + tygo 生成"，前端字段名永远
  跟 Go tag 一致。
- **D-19（fenced block 协议 dag/form/candidates/media）**：4 类 fenced block
  的 schema 都在 `internal/wire/fenced.go`，前端共用同一份 TS interface
  渲染。
- **D-20（skill = cc plugin overlay）**：`tether.toml` schema 一样可以用
  Go struct + tygo 生成 TS（即便 Go 端只读、不写 TOML）。

## 5. 风险 / 注意点

1. **生成产物入 git 还是不入**：建议**入 git**（`web/src/lib/wire.gen.ts`），
   理由：CI 跑 `tygo generate` 后 diff 检查能立刻发现"忘了跑 codegen"
   问题。如果不入，前后端可能 import 出不存在的文件。
2. **tygo 不支持 generic type**：v0.1 wire types 全是简单 struct，没
   generic，不是问题。如果 v0.2 要用，加 `// tygo:typename ...` 注释
   标记替代名。
3. **`json.RawMessage` 处理**：tygo 默认翻成 `unknown`，TS 这边需要
   手动 narrow。可以接受。
4. **`time.Time` 跨语言**：Go 端 marshal 成 ISO 8601 字符串，TS 端
   `Date | string` 都能用。type_mappings 设成 `string` 让 TS 端拿到
   就是 string，需要 `new Date(...)` 自己解析。

## 6. Acceptance criteria

- [ ] `internal/wire/` 包存在，至少有 `types.go` + `format.go` + `doc.go`
- [ ] `tygo.yaml` 在 repo 根（或 `internal/wire/`）
- [ ] `make codegen` 可执行，跑完产生 `web/src/lib/wire.gen.ts`
- [ ] `web/src/lib/wire.gen.ts` 入 git，`make codegen` 后 git diff 干净
- [ ] CI 在 build 之前运行 `make codegen` + `git diff --exit-code`
- [ ] `web/test/wire-contract.spec.ts` 覆盖以下 **6 条权威 wire 约定**（v2 §10.K.7 引用本列表，不再独立列）：
  1. **cert-hash format** — `/cert-hash` 跟 `/cert-hash-spki` 返回 64-char raw hex（`/^[0-9a-f]{64}$/`），无冒号无前缀
  2. **WT bidi pure echo** — 测试 endpoint 返回字节 = 输入字节（无 `echo: ` 前缀）
  3. **Envelope shape** — `wire.Envelope` JSON 字段名 (`kind` / `sessionId` / `payload`) 跟 Go struct json tag 一致
  4. **SessionID format** — UUIDv4 canonical lowercase (`/^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/`)
  5. **FencedBlockKind 枚举** — 仅 4 值：`"dag" | "form" | "candidates" | "media"`
  6. **tool_use input shape** — `wire.ToolUseEnvelope.input` 是 cc stream-json 直接 forward，键名/嵌套保持原样（`command` / `description` 等）

未来加新 wire 约定 → 在本列表加第 7、8 条。**不要在 v2 §10.K.7 重复列举**——那里只 reference 这一节。
- [ ] CI 在 build 之后运行 contract test（需要先起 server）
