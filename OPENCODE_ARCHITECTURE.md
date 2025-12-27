# OpenCode Architecture Document

> **Generated**: December 2024
> **Repository**: OpenCode - An AI-powered coding assistant CLI
> **Runtime**: TypeScript on Bun

---

## Table of Contents

1. [Conceptual Model](#1-conceptual-model)
2. [Codebase Architecture](#2-codebase-architecture)
3. [Low-Level Implementation Details](#3-low-level-implementation-details)
4. [If I Had to Modify This](#4-if-i-had-to-modify-this)
5. [Unknowns and Gaps](#5-unknowns-and-gaps)

---

## 1. Conceptual Model

### 1.1 The Agentic Loop

OpenCode implements a **streaming agentic loop** that continuously processes user prompts, invokes LLM completions, executes tools, and iterates until the model signals completion.

**Core Loop Flow:**

```
User Input → SessionPrompt.prompt() → SessionPrompt.loop() → SessionProcessor.process()
                                              ↓
                                    ┌─────────────────────┐
                                    │  LLM Streaming Call │
                                    └─────────────────────┘
                                              ↓
                                    ┌─────────────────────┐
                                    │  Tool Execution     │
                                    │  (if tool_use)      │
                                    └─────────────────────┘
                                              ↓
                                    ┌─────────────────────┐
                                    │  Continue Loop?     │──→ Yes: iterate
                                    │  (toolCalls.length) │
                                    └─────────────────────┘
                                              ↓ No
                                           Return
```

**Key implementation**: `packages/opencode/src/session/prompt.ts:95-141`

The loop terminates when:
1. No tool calls are returned by the model
2. An abort signal is received
3. The session is locked by another process

### 1.2 Memory System

OpenCode uses a **dual-layer memory architecture**:

| Layer | Purpose | Implementation |
|-------|---------|----------------|
| **Short-term** | Conversation messages within a session | In-memory array, persisted to SQLite |
| **Long-term** | Session history, share links, installation state | SQLite database via `better-sqlite3` |

**Memory Compaction** (`packages/opencode/src/session/compaction.ts`):
- Triggers when token count exceeds `PRUNE_MINIMUM` (20,000 tokens)
- Protects recent messages up to `PRUNE_PROTECT` (40,000 tokens)
- Uses LLM to generate summaries of pruned content
- Replaces verbose tool outputs with "[content pruned, see summary]"

```typescript
// Compaction constants (compaction.ts:6-7)
const PRUNE_MINIMUM = 20_000
const PRUNE_PROTECT = 40_000
```

### 1.3 Tool System

Tools are first-class citizens defined via a registry pattern:

```typescript
// Tool definition interface (tool/tool.ts:44-69)
Tool.define<Parameters, Metadata>(
  id: string,
  init: (ctx?) => Promise<{
    description: string
    parameters: ZodSchema
    execute(args, ctx): Promise<{ title, metadata, output, attachments? }>
  }>
)
```

**Tool Categories:**
- **Built-in tools**: File operations, shell execution, web fetch, etc.
- **MCP tools**: External tools via Model Context Protocol
- **Agent-specific tools**: Tools conditionally loaded based on active agent

### 1.4 Subagent / Delegation Pattern

OpenCode supports **subagent delegation** via the `Task` tool (`packages/opencode/src/tool/task.ts`):

- Creates isolated child sessions with their own conversation context
- Child agents can have different capabilities/tools than parent
- Results are returned to parent session as tool output
- Uses same LLM provider as parent session

**Subagent Types** (defined in `packages/opencode/src/agent/agent.ts`):
- `coder`: Primary coding agent with full tool access
- `task`: Background task execution agent
- `title`: Generates session titles
- `summarizer`: Creates conversation summaries

### 1.5 Prompt Construction

System prompts are assembled from multiple sources (`packages/opencode/src/session/system.ts` and `packages/opencode/src/session/llm.ts`):

```typescript
// Prompt assembly in LLM.stream() (llm.ts:49-61)
const system = SystemPrompt.header(input.model.providerID)
system.push(
  [
    // Agent prompt OR provider-specific prompt
    ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
    // Custom prompts passed into this call (environment, custom rules)
    ...input.system,
    // User-specified system prompt override
    ...(input.user.system ? [input.user.system] : []),
  ]
    .filter((x) => x)
    .join("\n"),
)
```

### 1.5.1 Prompt Storage Locations

All prompts are stored as `.txt` files imported at build time:

| Category | Location | Purpose |
|----------|----------|---------|
| **Main System Prompts** | `src/session/prompt/*.txt` | Model-specific base instructions |
| **Agent Prompts** | `src/agent/prompt/*.txt` | Specialized agent behaviors |
| **Tool Descriptions** | `src/tool/*.txt` | Per-tool usage instructions |
| **Command Templates** | `src/command/template/*.txt` | Slash command templates |

**Main System Prompt Files** (`session/prompt/`):

| File | Used For | Selection Logic |
|------|----------|-----------------|
| `anthropic.txt` | Claude models | `model.api.id.includes("claude")` |
| `anthropic_spoof.txt` | Header for Anthropic | Always prepended for anthropic provider |
| `gemini.txt` | Google Gemini models | `model.api.id.includes("gemini-")` |
| `beast.txt` | GPT-4, o1, o3 models | `model.api.id.includes("gpt-")` or `o1/o3` |
| `codex.txt` | GPT-5 models | `model.api.id.includes("gpt-5")` |
| `qwen.txt` | Fallback/other models | Default when no match |

### 1.5.2 Actual Prompt Content

**Main System Prompt** (`anthropic.txt`, ~106 lines):
```
You are OpenCode, the best coding agent on the planet.

You are an interactive CLI tool that helps users with software engineering tasks...

# Tone and style
- Only use emojis if the user explicitly requests it...
- Your output will be displayed on a command line interface...

# Professional objectivity
Prioritize technical accuracy and truthfulness over validating the user's beliefs...

# Task Management
You have access to the TodoWrite tools to help you manage and plan tasks...

# Doing tasks
The user will primarily request you perform software engineering tasks...

# Tool usage policy
- When doing file search, prefer to use the Task tool...
- You can call multiple tools in a single response...

# Code References
When referencing specific functions or pieces of code include the pattern
`file_path:line_number`...
```

**Agent-Specific Prompts**:

| Agent | Prompt File | Content Summary |
|-------|-------------|-----------------|
| `explore` | `agent/prompt/explore.txt` | "You are a file search specialist. You excel at thoroughly navigating and exploring codebases..." |
| `compaction` | `agent/prompt/compaction.txt` | "You are a helpful AI assistant tasked with summarizing conversations..." |
| `title` | `agent/prompt/title.txt` | "You are a title generator. You output ONLY a thread title..." with rules and examples |
| `summary` | `agent/prompt/summary.txt` | "You are a helpful AI assistant tasked with summarizing conversations..." |

**Tool Description Prompts** (examples):

| Tool | Description (from `.txt` file) |
|------|-------------------------------|
| `read` | "Reads a file from the local filesystem. You can access any file directly..." |
| `write` | "Writes a file to the local filesystem. This tool will overwrite the existing file..." |
| `edit` | "Performs exact string replacements in files. You must use your Read tool at least once..." |
| `bash` | "Executes a given bash command in a persistent shell session with optional timeout..." (includes git commit/PR guidelines) |
| `glob` | "Fast file pattern matching tool that works with any codebase size..." |
| `grep` | "A powerful search tool built on ripgrep..." |
| `task` | "Launch a new agent to handle complex, multi-step tasks autonomously..." |

### 1.5.3 Dynamic Prompt Assembly

The full system prompt sent to the LLM is assembled in this order (`llm.ts:49-74`):

```
┌─────────────────────────────────────────────────────────────┐
│ 1. HEADER (anthropic_spoof.txt for Anthropic providers)    │
├─────────────────────────────────────────────────────────────┤
│ 2. MAIN PROMPT (one of):                                    │
│    - Agent-specific prompt (if agent.prompt exists)         │
│    - OR Provider-specific prompt (anthropic.txt, etc.)      │
├─────────────────────────────────────────────────────────────┤
│ 3. ENVIRONMENT INFO (system.ts:35-58):                      │
│    <env>                                                    │
│      Working directory: /path/to/project                    │
│      Is directory a git repo: yes                           │
│      Platform: darwin                                       │
│      Today's date: Sat Dec 28 2024                          │
│    </env>                                                   │
│    <files>                                                  │
│      [200-line file tree from ripgrep]                      │
│    </files>                                                 │
├─────────────────────────────────────────────────────────────┤
│ 4. CUSTOM INSTRUCTIONS (system.ts:60-117):                  │
│    - AGENTS.md (local or global)                            │
│    - CLAUDE.md (~/.claude/CLAUDE.md)                        │
│    - config.instructions paths                              │
├─────────────────────────────────────────────────────────────┤
│ 5. USER SYSTEM OVERRIDE (if user.system specified)         │
└─────────────────────────────────────────────────────────────┘
```

**Custom Instruction File Priority** (`system.ts:60-68`):
```typescript
const LOCAL_RULE_FILES = ["AGENTS.md", "CLAUDE.md", "CONTEXT.md"]
const GLOBAL_RULE_FILES = [
  path.join(Global.Path.config, "AGENTS.md"),
  path.join(os.homedir(), ".claude", "CLAUDE.md"),
]
```

### 1.5.4 Tool Descriptions in Context

Tools are registered with descriptions from their `.txt` files and converted to LLM tool definitions:

```typescript
// From prompt.ts:572-648 (resolveTools)
tools[item.id] = tool({
  id: item.id,
  description: item.description,  // From tool's .txt file
  inputSchema: jsonSchema(schema),
  execute: async (args, options) => { ... }
})
```

The tool descriptions guide the LLM on when/how to use each tool. For example, `bash.txt` includes:
- When to use vs. specialized tools
- Directory verification steps
- Git safety protocols
- PR creation guidelines

### 1.6 Model Provider Abstraction

OpenCode uses **Vercel AI SDK** (`ai` package) as its provider abstraction layer:

```typescript
// Provider registry (provider/provider.ts)
createProviderRegistry({
  anthropic: createAnthropic(),
  openai: createOpenAI(),
  google: createGoogleGenerativeAI(),
  // ... additional providers
})
```

**Supported Providers:**
| Provider | Package | Notes |
|----------|---------|-------|
| Anthropic | `@ai-sdk/anthropic` | Primary/default |
| OpenAI | `@ai-sdk/openai` | GPT-4, GPT-4o |
| Google | `@ai-sdk/google` | Gemini models |
| AWS Bedrock | `@ai-sdk/amazon-bedrock` | Claude on AWS |
| Azure | `@ai-sdk/azure` | OpenAI on Azure |
| xAI | `@ai-sdk/xai` | Grok models |
| Groq | `@ai-sdk/groq` | Fast inference |

### 1.7 Safety Rails and Termination

**Abort Handling:**
- Every tool context receives an `AbortSignal` (`tool/tool.ts:18`)
- Sessions can be aborted via `Session.abort()` API
- Graceful shutdown on SIGINT/SIGTERM

**Rate Limiting & Retries:**
- Provider-level rate limiting handled by Vercel AI SDK
- Exponential backoff on transient errors
- Maximum retry count configurable per provider

### 1.8 End-to-End Walkthrough: Simple Request

**User types: "Create a hello world function"**

1. **CLI receives input** (`cli/cmd/run.ts`)
   - Creates or resumes session
   - Calls `SessionPrompt.prompt(input)`

2. **Message stored** (`session/prompt.ts:52-68`)
   - User message persisted to SQLite
   - `SessionPrompt.loop()` initiated

3. **System prompt assembled** (`session/system.ts`)
   - Agent instructions + tools + context combined

4. **LLM call** (`session/llm.ts`)
   - `streamText()` called with full message history
   - Response streamed back

5. **Response processed** (`session/processor.ts`)
   - Text parts extracted and displayed
   - Tool calls identified

6. **Tool execution** (if any)
   - Tools executed in parallel where possible
   - Results added to message history
   - Loop continues from step 4

7. **Completion**
   - Final response with no tool calls
   - Assistant message persisted
   - Control returned to CLI

### 1.9 Detailed Walkthrough: Complex Multi-Step Request

**User types: "Create a new user profile page with a REST API endpoint and database table"**

This example traces exactly how OpenCode processes a complex request requiring multiple tools and iterations.

---

#### Step 1: Input Reception and Session Setup

```
User Input: "Create a new user profile page with a REST API endpoint and database table"
```

**Code Path**: `cli/cmd/run.ts` → `SessionPrompt.prompt()`

```typescript
// prompt.ts:140-152
export const prompt = fn(PromptInput, async (input) => {
  const session = await Session.get(input.sessionID)
  await SessionRevert.cleanup(session)

  const message = await createUserMessage(input)  // Persist user message
  await Session.touch(input.sessionID)

  return loop(input.sessionID)  // Enter agentic loop
})
```

**User Message Created**:
```typescript
{
  id: "msg_01ABC...",
  role: "user",
  sessionID: "ses_01XYZ...",
  agent: "build",
  model: { providerID: "anthropic", modelID: "claude-sonnet-4-20250514" },
  time: { created: 1703788800000 }
}
```

---

#### Step 2: System Prompt Assembly

**Code Path**: `LLM.stream()` in `llm.ts:35-188`

The system prompt is assembled from multiple sources:

```typescript
// llm.ts:49-61
const system = SystemPrompt.header("anthropic")  // anthropic_spoof.txt
system.push([
  SystemPrompt.provider(model),  // anthropic.txt (main instructions)
  ...input.system,               // Environment + custom rules
].join("\n"))
```

**Final System Prompt Structure** (sent to Claude):

```
[SYSTEM MESSAGE 1 - Header]
<content from anthropic_spoof.txt - provider-specific header>

[SYSTEM MESSAGE 2 - Main + Environment]
You are OpenCode, the best coding agent on the planet.

You are an interactive CLI tool that helps users with software engineering tasks...

# Tone and style
- Only use emojis if the user explicitly requests it...
...

# Task Management
You have access to the TodoWrite tools to help you manage and plan tasks...
...

# Tool usage policy
- When doing file search, prefer to use the Task tool...
- You can call multiple tools in a single response...
...

Here is some useful information about the environment you are running in:
<env>
  Working directory: /home/user/myproject
  Is directory a git repo: yes
  Platform: linux
  Today's date: Sat Dec 28 2024
</env>
<files>
  src/
  ├── index.ts
  ├── routes/
  │   └── api.ts
  ├── db/
  │   └── schema.ts
  └── pages/
      └── home.tsx
</files>

Instructions from: /home/user/myproject/AGENTS.md
Use TypeScript for all new files. Follow existing patterns in the codebase.
```

---

#### Step 3: First LLM Call (Planning)

**Code Path**: `LLM.stream()` → `streamText()` from Vercel AI SDK

```typescript
// llm.ts:116-187
return streamText({
  model: wrapLanguageModel({ model: language, ... }),
  messages: [
    ...system.map(x => ({ role: "system", content: x })),
    ...input.messages,  // User message
  ],
  tools,  // All enabled tools with descriptions
  temperature: 0.7,
  maxOutputTokens: 32000,
  ...
})
```

**LLM Response (Iteration 1)**:
```
I'll help you create a user profile page with a REST API endpoint and database table.
Let me break this down into steps and start by exploring your existing codebase structure.

[TOOL_CALL: todowrite]
{
  "todos": [
    {"content": "Explore existing database schema", "status": "in_progress"},
    {"content": "Create user_profiles database table", "status": "pending"},
    {"content": "Create REST API endpoint for user profiles", "status": "pending"},
    {"content": "Create user profile page component", "status": "pending"}
  ]
}

[TOOL_CALL: glob]
{
  "pattern": "src/db/**/*.ts"
}

[TOOL_CALL: glob]
{
  "pattern": "src/routes/**/*.ts"
}
```

---

#### Step 4: Tool Execution (Parallel)

**Code Path**: `processor.ts:99-230` handles tool calls

```typescript
// processor.ts:122-180 (tool-call event)
case "tool-call": {
  const match = toolcalls[value.toolCallId]
  if (match) {
    await Session.updatePart({
      ...match,
      state: {
        status: "running",
        input: value.input,
        time: { start: Date.now() },
      },
    })
  }
  break
}
```

**Tools Executed in Parallel**:

1. **TodoWrite** → Creates todo list, returns: `"Todo list updated with 4 items"`

2. **Glob** (`src/db/**/*.ts`) → Returns:
   ```
   src/db/schema.ts
   src/db/connection.ts
   src/db/migrations/001_initial.ts
   ```

3. **Glob** (`src/routes/**/*.ts`) → Returns:
   ```
   src/routes/api.ts
   src/routes/auth.ts
   ```

**Tool Results Persisted** (`processor.ts:182-203`):
```typescript
case "tool-result": {
  await Session.updatePart({
    ...match,
    state: {
      status: "completed",
      input: value.input,
      output: value.output.output,
      time: { start: match.state.time.start, end: Date.now() },
    },
  })
}
```

---

#### Step 5: Loop Continues (Iteration 2)

**Code Path**: `prompt.ts:242-273` - loop check

```typescript
// prompt.ts:266-273
if (
  lastAssistant?.finish &&
  !["tool-calls", "unknown"].includes(lastAssistant.finish) &&
  lastUser.id < lastAssistant.id
) {
  break  // Exit loop if no more tool calls
}
// Continue looping...
```

Since `lastAssistant.finish === "tool-calls"`, the loop continues.

**LLM Response (Iteration 2)** - Reading existing files:
```
Let me examine the existing schema and routes to understand the patterns used.

[TOOL_CALL: read]
{
  "filePath": "/home/user/myproject/src/db/schema.ts"
}

[TOOL_CALL: read]
{
  "filePath": "/home/user/myproject/src/routes/api.ts"
}
```

**Read Results**:
```typescript
// src/db/schema.ts
import { pgTable, text, timestamp, uuid } from "drizzle-orm/pg-core"

export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: text("email").notNull().unique(),
  createdAt: timestamp("created_at").defaultNow(),
})
```

```typescript
// src/routes/api.ts
import { Hono } from "hono"
import { db } from "../db/connection"

const api = new Hono()

api.get("/users", async (c) => {
  const users = await db.select().from(users)
  return c.json(users)
})

export default api
```

---

#### Step 6: Loop Continues (Iteration 3) - Writing Files

**LLM Response (Iteration 3)**:
```
Now I understand the patterns. Let me create the user profile table, API endpoint, and page.

[TOOL_CALL: todowrite]
{
  "todos": [
    {"content": "Explore existing database schema", "status": "completed"},
    {"content": "Create user_profiles database table", "status": "in_progress"},
    {"content": "Create REST API endpoint for user profiles", "status": "pending"},
    {"content": "Create user profile page component", "status": "pending"}
  ]
}

[TOOL_CALL: edit]
{
  "filePath": "/home/user/myproject/src/db/schema.ts",
  "oldString": "export const users = pgTable(\"users\", {",
  "newString": "export const userProfiles = pgTable(\"user_profiles\", {\n  id: uuid(\"id\").primaryKey().defaultRandom(),\n  userId: uuid(\"user_id\").references(() => users.id),\n  displayName: text(\"display_name\"),\n  bio: text(\"bio\"),\n  avatarUrl: text(\"avatar_url\"),\n  updatedAt: timestamp(\"updated_at\").defaultNow(),\n})\n\nexport const users = pgTable(\"users\", {"
}
```

---

#### Step 7: More Iterations (File Creation)

**LLM Response (Iteration 4)** - Creating API endpoint:
```
[TOOL_CALL: edit]
{
  "filePath": "/home/user/myproject/src/routes/api.ts",
  "oldString": "export default api",
  "newString": "// User Profile endpoints\napi.get(\"/profiles/:userId\", async (c) => {\n  const userId = c.req.param(\"userId\")\n  const profile = await db.select().from(userProfiles).where(eq(userProfiles.userId, userId))\n  return c.json(profile[0] ?? null)\n})\n\napi.put(\"/profiles/:userId\", async (c) => {\n  const userId = c.req.param(\"userId\")\n  const body = await c.req.json()\n  const updated = await db.update(userProfiles).set(body).where(eq(userProfiles.userId, userId)).returning()\n  return c.json(updated[0])\n})\n\nexport default api"
}
```

**LLM Response (Iteration 5)** - Creating page component:
```
[TOOL_CALL: write]
{
  "filePath": "/home/user/myproject/src/pages/profile.tsx",
  "content": "import { useState, useEffect } from 'react'\n\ninterface UserProfile {\n  id: string\n  userId: string\n  displayName: string | null\n  bio: string | null\n  avatarUrl: string | null\n}\n\nexport function ProfilePage({ userId }: { userId: string }) {\n  const [profile, setProfile] = useState<UserProfile | null>(null)\n  const [loading, setLoading] = useState(true)\n\n  useEffect(() => {\n    fetch(`/api/profiles/${userId}`)\n      .then(res => res.json())\n      .then(data => {\n        setProfile(data)\n        setLoading(false)\n      })\n  }, [userId])\n\n  if (loading) return <div>Loading...</div>\n  if (!profile) return <div>Profile not found</div>\n\n  return (\n    <div className=\"profile-page\">\n      <img src={profile.avatarUrl ?? '/default-avatar.png'} alt=\"Avatar\" />\n      <h1>{profile.displayName ?? 'Anonymous'}</h1>\n      <p>{profile.bio}</p>\n    </div>\n  )\n}\n"
}

[TOOL_CALL: todowrite]
{
  "todos": [
    {"content": "Explore existing database schema", "status": "completed"},
    {"content": "Create user_profiles database table", "status": "completed"},
    {"content": "Create REST API endpoint for user profiles", "status": "completed"},
    {"content": "Create user profile page component", "status": "completed"}
  ]
}
```

---

#### Step 8: Final Response (No Tool Calls)

**LLM Response (Iteration 6)** - Completion:
```
I've created the user profile feature with the following components:

1. **Database Table** (`src/db/schema.ts:1-10`):
   - Added `userProfiles` table with fields for displayName, bio, and avatarUrl
   - References the existing users table via userId

2. **REST API Endpoints** (`src/routes/api.ts:15-28`):
   - `GET /api/profiles/:userId` - Fetch a user's profile
   - `PUT /api/profiles/:userId` - Update a user's profile

3. **Profile Page Component** (`src/pages/profile.tsx`):
   - React component that fetches and displays user profile data
   - Handles loading and not-found states

To complete the setup, you'll need to:
1. Run a database migration: `bun run db:migrate`
2. Import the ProfilePage in your router
```

**Loop Exit** (`prompt.ts:266-273`):
- `lastAssistant.finish === "stop"` (no tool calls)
- Loop breaks, final message persisted

---

#### Step 9: Message Persistence

All messages and tool calls are persisted to SQLite:

```typescript
// Session.updateMessage() and Session.updatePart() called throughout
// Final state in database:

Session: {
  id: "ses_01XYZ...",
  title: "Creating user profile feature",  // Generated by title agent
  agent: "build",
  ...
}

Messages: [
  { role: "user", content: "Create a new user profile page..." },
  { role: "assistant", parts: [
    { type: "text", text: "I'll help you create..." },
    { type: "tool", tool: "todowrite", state: { status: "completed", ... } },
    { type: "tool", tool: "glob", state: { status: "completed", ... } },
    ...
  ]},
  { role: "assistant", parts: [
    { type: "text", text: "Let me examine..." },
    { type: "tool", tool: "read", ... },
    ...
  ]},
  // ... more iterations
  { role: "assistant", parts: [
    { type: "text", text: "I've created the user profile feature..." },
  ]},
]
```

---

#### Sequence Diagram: Complex Request Flow

```mermaid
sequenceDiagram
    participant User
    participant CLI
    participant Prompt as SessionPrompt
    participant LLM
    participant Tools

    User->>CLI: "Create user profile page with API and DB table"
    CLI->>Prompt: prompt(input)
    Prompt->>Prompt: createUserMessage()
    Prompt->>Prompt: loop()

    rect rgb(240, 248, 255)
        Note over Prompt,Tools: Iteration 1: Planning
        Prompt->>LLM: streamText(system + user message)
        LLM-->>Prompt: text + todowrite + glob×2
        Prompt->>Tools: execute(todowrite, glob, glob) [parallel]
        Tools-->>Prompt: results
    end

    rect rgb(255, 248, 240)
        Note over Prompt,Tools: Iteration 2: Reading
        Prompt->>LLM: streamText(+ tool results)
        LLM-->>Prompt: text + read×2
        Prompt->>Tools: execute(read, read) [parallel]
        Tools-->>Prompt: file contents
    end

    rect rgb(240, 255, 240)
        Note over Prompt,Tools: Iteration 3-5: Writing
        Prompt->>LLM: streamText(+ tool results)
        LLM-->>Prompt: text + edit
        Prompt->>Tools: execute(edit)
        Tools-->>Prompt: success
        Note over Prompt,LLM: Repeat for API and component
    end

    rect rgb(255, 255, 240)
        Note over Prompt,Tools: Iteration 6: Completion
        Prompt->>LLM: streamText(+ tool results)
        LLM-->>Prompt: text only (no tools)
    end

    Prompt-->>CLI: final message
    CLI-->>User: Display response
```

---

#### Summary: What Makes This Work

1. **Prompt Construction** guides the LLM with:
   - Clear role definition ("best coding agent")
   - Task management instructions (use TodoWrite)
   - Tool usage policies (parallel calls, specialized tools)
   - Environment context (file tree, git status)

2. **Agentic Loop** (`prompt.ts:230-563`) enables:
   - Unlimited iterations until completion
   - Tool call detection and execution
   - Automatic continuation after tool results

3. **Tool System** provides:
   - File operations (read, write, edit, glob)
   - Task tracking (todowrite)
   - Code search (grep, glob)
   - Shell access (bash)

4. **Streaming** (`processor.ts`) handles:
   - Real-time output display
   - Concurrent tool execution
   - Error recovery and retries

---

## 2. Codebase Architecture

### 2.1 Repository Structure

```
opencode/
├── packages/
│   └── opencode/                    # Main application package
│       ├── bin/opencode             # CLI entrypoint (shebang script)
│       ├── src/
│       │   ├── index.ts             # Public API exports
│       │   ├── agent/               # Agent definitions and management
│       │   ├── app/                 # TUI application (Ink/React)
│       │   ├── bus/                 # Event pub/sub system
│       │   ├── cli/                 # CLI commands and argument parsing
│       │   ├── config/              # Configuration management
│       │   ├── file/                # File system operations
│       │   ├── lsp/                 # Language Server Protocol client
│       │   ├── mcp/                 # Model Context Protocol integration
│       │   ├── permission/          # Permission system
│       │   ├── project/             # Project context management
│       │   ├── provider/            # LLM provider abstraction
│       │   ├── server/              # HTTP API server (Hono)
│       │   ├── session/             # Core session/conversation logic
│       │   ├── skill/               # Skill/plugin system
│       │   ├── storage/             # SQLite persistence layer
│       │   ├── tool/                # Tool registry and definitions
│       │   └── util/                # Shared utilities
│       └── drizzle/                 # Database migrations
├── www/                             # Documentation website
└── bun.lockb                        # Bun lockfile
```

### 2.2 Component Responsibilities

| Component | Primary File(s) | Responsibility |
|-----------|-----------------|----------------|
| **Session Orchestrator** | `session/prompt.ts` | Runs agentic loop, coordinates LLM calls and tool execution |
| **LLM Integration** | `session/llm.ts` | Wraps Vercel AI SDK for streaming/non-streaming calls |
| **Tool Registry** | `tool/registry.ts` | Discovers and initializes available tools |
| **Storage Layer** | `storage/storage.ts` | SQLite operations, migrations, queries |
| **Provider Manager** | `provider/provider.ts` | Model provider abstraction and selection |
| **Event Bus** | `bus/index.ts` | Instance-scoped pub/sub for events |
| **Project Context** | `project/instance.ts` | Manages per-directory instance state |
| **Configuration** | `config/config.ts` | Loads and validates user configuration |
| **CLI Interface** | `cli/cmd/*.ts` | Command parsing and execution |
| **HTTP Server** | `server/server.ts` | REST API for remote/programmatic access |

### 2.3 C4 Context Diagram

```mermaid
C4Context
    title OpenCode System Context

    Person(user, "Developer", "Uses OpenCode for AI-assisted coding")

    System(opencode, "OpenCode", "AI-powered coding assistant CLI")

    System_Ext(llm_providers, "LLM Providers", "Anthropic, OpenAI, Google, etc.")
    System_Ext(mcp_servers, "MCP Servers", "External tool providers via MCP")
    System_Ext(filesystem, "File System", "Project files and directories")
    System_Ext(shell, "Shell", "Terminal command execution")
    System_Ext(lsp_servers, "LSP Servers", "Language intelligence")

    Rel(user, opencode, "Interacts via CLI/TUI")
    Rel(opencode, llm_providers, "Sends prompts, receives completions", "HTTPS")
    Rel(opencode, mcp_servers, "Tool invocations", "stdio/HTTPS")
    Rel(opencode, filesystem, "Reads/writes files")
    Rel(opencode, shell, "Executes commands")
    Rel(opencode, lsp_servers, "Code intelligence queries", "JSON-RPC")
```

### 2.4 C4 Container Diagram

```mermaid
C4Container
    title OpenCode Container Diagram

    Person(user, "Developer")

    Container_Boundary(opencode, "OpenCode Application") {
        Container(cli, "CLI Interface", "TypeScript/Ink", "Command parsing, TUI rendering")
        Container(session, "Session Manager", "TypeScript", "Agentic loop orchestration")
        Container(tools, "Tool System", "TypeScript", "Tool registry and execution")
        Container(storage, "Storage Layer", "SQLite/better-sqlite3", "Persistence")
        Container(server, "HTTP Server", "Hono", "REST API for programmatic access")
        Container(providers, "Provider Layer", "Vercel AI SDK", "LLM abstraction")
    }

    System_Ext(anthropic, "Anthropic API")
    System_Ext(openai, "OpenAI API")
    System_Ext(filesystem, "File System")

    Rel(user, cli, "Interacts")
    Rel(cli, session, "Creates sessions, sends prompts")
    Rel(session, tools, "Executes tools")
    Rel(session, providers, "LLM calls")
    Rel(session, storage, "Persists messages")
    Rel(tools, filesystem, "File operations")
    Rel(providers, anthropic, "API calls")
    Rel(providers, openai, "API calls")
    Rel(server, session, "Programmatic session control")
```

### 2.5 Component Diagram

```mermaid
C4Component
    title Session Manager Components

    Container_Boundary(session, "Session Manager") {
        Component(prompt, "SessionPrompt", "Orchestrates the agentic loop")
        Component(processor, "SessionProcessor", "Processes LLM responses")
        Component(llm, "SessionLLM", "LLM API integration")
        Component(system, "SystemPrompt", "Prompt construction")
        Component(compaction, "Compaction", "Memory management")
        Component(messages, "MessageV2", "Message type definitions")
    }

    Rel(prompt, processor, "Delegates processing")
    Rel(processor, llm, "Calls LLM")
    Rel(llm, system, "Gets system prompt")
    Rel(prompt, compaction, "Triggers when needed")
    Rel(processor, messages, "Creates/transforms")
```

### 2.6 Sequence Diagram: User Prompt Flow

```mermaid
sequenceDiagram
    participant User
    participant CLI
    participant Session
    participant SessionPrompt
    participant SessionProcessor
    participant LLM
    participant ToolRegistry
    participant Tool
    participant Storage

    User->>CLI: Enter prompt
    CLI->>Session: create/resume session
    CLI->>SessionPrompt: prompt(input)
    SessionPrompt->>Storage: persist user message
    SessionPrompt->>SessionPrompt: loop()

    loop Agentic Loop
        SessionPrompt->>SessionProcessor: process()
        SessionProcessor->>LLM: streamText()
        LLM-->>SessionProcessor: streaming response

        alt Has Tool Calls
            SessionProcessor->>ToolRegistry: get tool
            ToolRegistry-->>SessionProcessor: tool instance
            SessionProcessor->>Tool: execute(args)
            Tool-->>SessionProcessor: result
            SessionProcessor->>Storage: persist tool result
        end

        SessionProcessor-->>SessionPrompt: {toolCalls, parts}
    end

    SessionPrompt->>Storage: persist assistant message
    SessionPrompt-->>CLI: complete
    CLI-->>User: Display response
```

### 2.7 Sequence Diagram: Tool Execution

```mermaid
sequenceDiagram
    participant Processor as SessionProcessor
    participant Registry as ToolRegistry
    participant Tool
    participant Permission
    participant Bus

    Processor->>Registry: tools.get(toolName)
    Registry-->>Processor: Tool.Info

    Processor->>Permission: check(toolName, args)
    Permission-->>Processor: allowed/denied

    alt Allowed
        Processor->>Tool: execute(args, context)
        Tool->>Tool: validate args (Zod)
        Tool->>Tool: perform operation
        Tool->>Bus: publish events (optional)
        Tool-->>Processor: {title, metadata, output}
    else Denied
        Processor-->>Processor: return permission error
    end
```

---

## 3. Low-Level Implementation Details

### 3.1 Entrypoints

| Entrypoint | File | Purpose |
|------------|------|---------|
| CLI Binary | `packages/opencode/bin/opencode` | Shebang script invoking Bun |
| CLI Commands | `packages/opencode/src/cli/cmd/*.ts` | Individual command implementations |
| HTTP Server | `packages/opencode/src/server/server.ts` | REST API entrypoint |
| SDK Export | `packages/opencode/src/index.ts` | Programmatic API |

**CLI Entrypoint** (`bin/opencode`):
```sh
#!/usr/bin/env bun
import "../src/cli"
```

**Main CLI Setup** (`src/cli/index.ts`):
- Uses `yargs` for argument parsing
- Registers commands: `run`, `config`, `share`, `mcp`, etc.

### 3.2 Orchestrator (Session Loop)

**File**: `packages/opencode/src/session/prompt.ts`

```typescript
// Core loop implementation (lines 95-141)
async function loop(sessionID: string, opts: LoopOptions) {
  while (true) {
    const locked = await Session.lock.acquire(sessionID)
    if (!locked) throw new Error("Session locked")

    try {
      const result = await SessionProcessor.process({ sessionID, ...opts })

      if (result.aborted) break
      if (!result.toolCalls.length) break  // No more tool calls = done

      // Continue loop with tool results
    } finally {
      Session.lock.release(sessionID)
    }
  }
}
```

**Key Functions:**
- `SessionPrompt.prompt()` - Entry point for user messages
- `SessionPrompt.loop()` - Main agentic loop
- `SessionPrompt.retry()` - Retry last failed operation
- `SessionPrompt.revert()` - Undo last assistant turn

### 3.3 Prompt Builder

**File**: `packages/opencode/src/session/system.ts`

The system prompt is constructed dynamically:

```typescript
// Prompt assembly (lines 30-70, approximate)
export async function build(opts: BuildOptions): Promise<string> {
  const parts: string[] = []

  // 1. Agent base instructions
  parts.push(agent.system)

  // 2. Active skills
  for (const skill of activeSkills) {
    parts.push(await skill.getPrompt())
  }

  // 3. Project context
  if (project.context) {
    parts.push(project.context)
  }

  // 4. Environment info
  parts.push(formatEnvironment())

  return parts.join("\n\n")
}
```

### 3.4 Tool Registry

**File**: `packages/opencode/src/tool/registry.ts`

```typescript
// Tool discovery and initialization
export const ToolRegistry = {
  async tools(ctx: InitContext): Promise<Map<string, Tool.Info>> {
    const tools = new Map()

    // Built-in tools
    for (const tool of builtinTools) {
      const initialized = await tool.init(ctx)
      tools.set(tool.id, initialized)
    }

    // MCP tools (external)
    const mcpTools = await MCP.getTools()
    for (const tool of mcpTools) {
      tools.set(tool.id, tool)
    }

    return tools
  }
}
```

**Built-in Tools** (from `tool/` directory):
| Tool | File | Description |
|------|------|-------------|
| `bash` | `bash.ts` | Shell command execution |
| `file_read` | `file.ts` | Read file contents |
| `file_write` | `file.ts` | Write/create files |
| `file_edit` | `edit.ts` | Edit file sections |
| `glob` | `glob.ts` | File pattern matching |
| `grep` | `grep.ts` | Content search |
| `task` | `task.ts` | Subagent delegation |
| `web_fetch` | `web.ts` | HTTP requests |

### 3.5 Memory / Persistence Layer

**File**: `packages/opencode/src/storage/storage.ts`

Uses `better-sqlite3` for synchronous SQLite operations:

```typescript
// Database initialization
const db = new Database(path.join(dataDir, "opencode.db"))
db.pragma("journal_mode = WAL")

// Run migrations
await migrate(db, migrationsFolder)
```

**Tables** (inferred from migrations in `drizzle/`):
- `sessions` - Session metadata
- `messages` - Conversation messages (JSON blob)
- `shares` - Shared session links
- `installations` - MCP server installations

**Message Storage Format** (`session/message-v2.ts`):
```typescript
interface MessageV2 {
  id: string
  role: "user" | "assistant" | "tool"
  parts: MessagePart[]
  metadata: { ... }
  createdAt: Date
}
```

### 3.6 LLM Provider Integration

**File**: `packages/opencode/src/provider/provider.ts`

```typescript
// Provider registry setup
export const registry = createProviderRegistry({
  anthropic: createAnthropic({ apiKey: config.anthropic?.apiKey }),
  openai: createOpenAI({ apiKey: config.openai?.apiKey }),
  google: createGoogleGenerativeAI({ apiKey: config.google?.apiKey }),
  // ... more providers
})

// Model resolution
export function getModel(modelId: string): LanguageModel {
  return registry.languageModel(modelId)
}
```

**LLM Call** (`session/llm.ts`):
```typescript
// Streaming call
const result = await streamText({
  model: getModel(modelId),
  messages: formattedMessages,
  system: systemPrompt,
  tools: toolDefinitions,
  maxTokens: config.maxTokens,
  temperature: config.temperature,
})

// Process stream
for await (const chunk of result.stream) {
  // Handle text deltas, tool calls, etc.
}
```

### 3.7 Key Data Types

**Session** (`session/index.ts`):
```typescript
interface Session.Info {
  id: string
  title: string
  agent: string
  modelId: string
  createdAt: Date
  updatedAt: Date
}
```

**Message** (`session/message-v2.ts`):
```typescript
interface MessageV2 {
  id: string
  sessionId: string
  role: "user" | "assistant"
  parts: (TextPart | ToolCallPart | ToolResultPart | FilePart)[]
  createdAt: Date
}
```

**Tool Context** (`tool/tool.ts:14-22`):
```typescript
interface Tool.Context {
  sessionID: string
  messageID: string
  agent: string
  abort: AbortSignal
  callID?: string
  extra?: Record<string, any>
  metadata(input: { title?: string; metadata?: any }): void
}
```

### 3.8 Error Handling

**Tool Validation Errors** (`tool/tool.ts:54-64`):
```typescript
try {
  toolInfo.parameters.parse(args)
} catch (error) {
  if (error instanceof z.ZodError && toolInfo.formatValidationError) {
    throw new Error(toolInfo.formatValidationError(error), { cause: error })
  }
  throw new Error(
    `The ${id} tool was called with invalid arguments: ${error}.`,
    { cause: error }
  )
}
```

**LLM Errors** - Handled by Vercel AI SDK with retry logic

**Abort Handling** - Propagated via `AbortSignal` through tool context

### 3.9 Logging and Observability

**File**: `packages/opencode/src/util/log.ts`

```typescript
// Structured logging
const log = Log.create({ service: "session" })
log.info("processing", { sessionId, messageCount })
log.error("tool failed", { toolId, error })
```

**Event Bus** (`bus/index.ts`) for internal observability:
```typescript
Bus.publish(SessionEvent.MessageCreated, { sessionId, messageId })
Bus.subscribe(SessionEvent.MessageCreated, (event) => { ... })
```

---

## 4. If I Had to Modify This

### 4.1 Adding a New Tool

1. **Create tool file**: `packages/opencode/src/tool/my-tool.ts`
```typescript
import { Tool } from "./tool"
import z from "zod"

export const MyTool = Tool.define(
  "my_tool",
  {
    description: "Does something useful",
    parameters: z.object({
      input: z.string().describe("The input to process"),
    }),
    async execute(args, ctx) {
      // Implementation
      return {
        title: "My Tool",
        metadata: {},
        output: `Processed: ${args.input}`,
      }
    },
  }
)
```

2. **Register in registry**: `packages/opencode/src/tool/registry.ts`
```typescript
import { MyTool } from "./my-tool"
// Add to builtinTools array
```

### 4.2 Adding a New LLM Provider

1. **Install SDK package**: Add to `packages/opencode/package.json`
```json
"@ai-sdk/new-provider": "^1.0.0"
```

2. **Register provider**: `packages/opencode/src/provider/provider.ts`
```typescript
import { createNewProvider } from "@ai-sdk/new-provider"

export const registry = createProviderRegistry({
  // ... existing providers
  newprovider: createNewProvider({ apiKey: config.newprovider?.apiKey }),
})
```

3. **Add config schema**: `packages/opencode/src/config/config.ts`
```typescript
newprovider: z.object({
  apiKey: z.string().optional(),
}).optional()
```

### 4.3 Adding a New Memory Backend

**Currently**: SQLite only (`storage/storage.ts`)

**To add alternative (e.g., PostgreSQL)**:

1. Create new storage adapter: `packages/opencode/src/storage/postgres.ts`
2. Implement same interface as SQLite storage
3. Add configuration option for storage backend selection
4. Update `storage/storage.ts` to delegate to selected backend

**Key interfaces to implement**:
- `sessions.create()`, `sessions.get()`, `sessions.list()`
- `messages.create()`, `messages.list()`, `messages.update()`
- `shares.create()`, `shares.get()`

### 4.4 Adding a New Agent Type

1. **Define agent**: `packages/opencode/src/agent/agent.ts`
```typescript
export const myAgent: Agent.Info = {
  id: "my-agent",
  name: "My Custom Agent",
  system: `You are a specialized agent for...`,
  tools: ["tool1", "tool2"],  // Allowed tools
}
```

2. **Register**: Add to agents array in same file

### 4.5 Modifying Prompt Construction

**File**: `packages/opencode/src/session/system.ts`

Key modification points:
- Line ~30: Add new prompt sections
- Skill integration: `packages/opencode/src/skill/`
- Agent-specific prompts: `packages/opencode/src/agent/agent.ts`

---

## 5. Unknowns and Gaps

### 5.1 Areas Requiring Further Investigation

| Area | Question | Files to Explore |
|------|----------|------------------|
| **MCP Protocol** | Full message format and handshake details | `src/mcp/` directory |
| **Permission System** | Exact permission rules and persistence | `src/permission/` |
| **LSP Integration** | How LSP is used for code intelligence | `src/lsp/` |
| **TUI Framework** | Ink/React component architecture | `src/app/` |
| **Share System** | How session sharing works | `src/share/` (if exists) |

### 5.2 Unverified Assumptions

1. **Retry behavior**: Assumed exponential backoff, but exact implementation not traced
2. **Token counting**: Assumed tiktoken or similar, but tokenizer not explicitly identified
3. **Streaming protocol**: Assumed SSE for HTTP server, not verified
4. **Concurrent tool execution**: Parallel execution mentioned but limits not identified

### 5.3 Missing from Analysis

- **Test coverage**: Test files not analyzed
- **Build/deployment**: CI/CD configuration not reviewed
- **Performance characteristics**: No profiling data
- **Security model**: Permission system not fully traced
- **Plugin/skill authoring**: Skill creation documentation not found

### 5.4 Potential Architectural Risks

1. **Single SQLite database**: May become bottleneck at scale
2. **In-memory message cache**: Large conversations could cause memory pressure
3. **Synchronous SQLite**: `better-sqlite3` is sync; could block event loop in edge cases
4. **Global state via Instance**: Instance pattern could complicate testing/parallelism

---

## Appendix: File Reference

| Category | Key Files |
|----------|-----------|
| Entrypoints | `bin/opencode`, `src/cli/index.ts`, `src/server/server.ts` |
| Core Loop | `src/session/prompt.ts`, `src/session/processor.ts` |
| LLM | `src/session/llm.ts`, `src/provider/provider.ts` |
| Tools | `src/tool/tool.ts`, `src/tool/registry.ts`, `src/tool/*.ts` |
| Storage | `src/storage/storage.ts`, `drizzle/*.sql` |
| Config | `src/config/config.ts` |
| Events | `src/bus/index.ts`, `src/bus/bus-event.ts` |
| Agents | `src/agent/agent.ts` |

---

*Document generated by analyzing the OpenCode repository. For the most current architecture, always refer to the source code.*
