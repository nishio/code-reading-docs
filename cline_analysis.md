# ClineとRooの実行フロー解析

## 要約

ClineとRoo-Codeは、VS Code向けのAIコーディングアシスタントとして開発された2つのプロジェクトです。本文書では、両プロジェクトの実装、進化、および特徴的な違いについて分析します。

### 主な特徴

1. アーキテクチャの違い
   - Cline: シンプルで直接的なモノリシック構造を採用し、基本機能の最適化に注力
   - Roo-Code: モジュール化された構造で、拡張性とカスタマイズ性を重視

2. プロンプト管理
   - Cline: 単一のシステムプロンプトで一貫した動作を実現
   - Roo-Code: モード別の動的プロンプト生成システムを実装

3. ツール管理
   - Cline: フラットな構造でシンプルな権限管理を実現
   - Roo-Code: グループベースの階層的なツール管理システムを採用

4. 開発の焦点
   - Cline: コア機能の強化、パフォーマンス最適化、モデル対応の拡充
   - Roo-Code: コミュニティ機能、UI/UX改善、セキュリティ強化

### 進化の過程

1. 初期実装（2024年7月）
   - VS Code拡張の基本設定
   - Webviewの実装とReact UIの統合
   - 基本的なメッセージング処理の実装

2. 機能拡張
   - ツールシステムの実装と権限管理の追加
   - MCPシステムの導入と拡張
   - モード管理システムの実装（Roo-Code）

3. 最近の開発動向
   - Cline: シェル検出機能の改善、ストリーミング処理の最適化
   - Roo-Code: Discord連携、ドキュメント整備、UI改善

## 1. プロジェクトの進化

### 1.1 初期の動作段階 (2024年7月7-8日)

1. 基本機能の実装:
- VS Code拡張機能の基本設定
- React UIの統合（cd8bbc5）：Webviewの実装とReactアプリケーションの統合
- Webviewとの双方向通信（991ea6b）：メッセージングシステムの実装

2. コア機能の実装:
- APIキー管理とグローバルステート（8ba1be1）：
  ```typescript
  // src/providers/SidebarProvider.ts
  private async storeSecret(key: ExtensionSecretKey, value: any) {
      await this.context.secrets.store(key, value)
  }
  ```
- システムプロンプトの初期実装（09559c3）：
  - Claude 3.5 Sonnetの機能を活用
  - ツール実行の権限管理
  - コンテキスト管理システム
- メッセージング処理の基本実装：
  ```typescript
  // src/ClaudeDev.ts
  async startTask(task: string): Promise<void> {
      const filesInCurrentDir = await this.listFiles(".")
      let userPrompt = `# Task\n${task}\n\n# Context\n...`
      await this.say("text", userPrompt)
  }
  ```

### 1.2 機能拡張段階 (2024年7月8-10日)

1. ツールシステムの実装:
```typescript
// src/ClaudeDev.ts
const tools: Tool[] = [
    {
        name: "execute_command",
        description: "Execute a CLI command on the system. Use this when you need to perform system operations or run specific commands to accomplish any step in the user's task.",
        input_schema: {
            type: "object",
            properties: {
                command: { type: "string" }
            },
            required: ["command"]
        }
    },
    {
        name: "list_files",
        description: "List all files and directories at the top level of the specified directory.",
        input_schema: {
            type: "object",
            properties: {
                path: { type: "string" }
            },
            required: ["path"]
        }
    },
    {
        name: "read_file",
        description: "Read the contents of a file at the specified path.",
        input_schema: {
            type: "object",
            properties: {
                path: { type: "string" }
            },
            required: ["path"]
        }
    }
]
```

2. 権限管理システムの追加（3d7abe4）:
- ツール実行時の承認フロー：
  ```typescript
  // src/ClaudeDev.ts
  async executeCommand(command: string): Promise<string> {
      const { response } = await this.ask("command", command)
      if (response === "yesButtonTapped") {
          // Execute command after user approval
          return this.runCommand(command)
      }
      return "Command execution cancelled by user"
  }
  ```
- セキュリティ制御：
  - ファイルアクセスの制限
  - コマンド実行の承認要求
  - APIキーの安全な管理
- ユーザーインターフェースの改善：
  - 承認ダイアログの実装
  - 実行状態の可視化
  - エラーハンドリングの強化

### 1.3 メッセージング処理の進化

1. システムプロンプトの実装:
```typescript
// src/core/prompts/system.ts
export const SYSTEM_PROMPT = async (
    cwd: string,
    supportsComputerUse: boolean,
) => `You are Cline, a highly skilled software engineer...
Available tools:
${tools.map(tool => `- ${tool.name}: ${tool.description}`).join('\n')}
...`
```

2. メッセージング処理の最適化:
```typescript
// src/core/Cline.ts
export class Cline {
    async initiateTaskLoop(messages: Message[], isFirstMessage: boolean = false): Promise<void> {
        try {
            const stream = await this.attemptApiRequest(this.apiRequestIndex)
            for await (const chunk of stream) {
                await this.handleApiResponse(chunk, isFirstMessage)
            }
        } catch (error) {
            await this.handleApiError(error)
        }
    }
}
```

3. エラー処理とリカバリー:
- エラー発生時の自動リトライ
- コンテキストウィンドウの管理
- メッセージ同期の最適化
- チェックポイントによる状態管理

## 2. 実行フロー

### 2.1 タスク開始フロー

1. ユーザーからの指示受信:
```typescript
// src/core/Cline.ts
async startTask(task?: string, images?: string[]): Promise<void> {
    this.clineMessages = []
    this.apiConversationHistory = []
    await this.say("text", task, images)
    this.isInitialized = true
    await this.initiateTaskLoop([
        {
            type: "text",
            text: `<task>\n${task}\n</task>`,
        },
    ], true)
}
```

2. コンテキスト収集:
```typescript
// src/core/Cline.ts
async *attemptApiRequest(previousApiReqIndex: number): ApiStream {
    let systemPrompt = await SYSTEM_PROMPT(
        cwd,
        this.api.getModel().info.supportsComputerUse ?? false,
        mcpHub,
        this.browserSettings,
    )
    const truncatedConversationHistory = getTruncatedMessages(
        this.apiConversationHistory,
        this.conversationHistoryDeletedRange,
    )
}
```

3. LLMへのプロンプト発行:
```typescript
// src/core/prompts/system.ts
export const SYSTEM_PROMPT = async (
    cwd: string,
    supportsComputerUse: boolean,
) => `You are Cline, a highly skilled software engineer...
Available tools:
${tools.map(tool => `- ${tool.name}: ${tool.description}`).join('\n')}

Current Working Directory: ${cwd}
...`
```

### 2.2 ツール実行フロー

1. ツールの選択と実行:
```typescript
// src/core/Cline.ts
async executeTool(toolName: string, toolInput: any): Promise<string> {
    switch (toolName) {
        case "execute_command":
            return this.executeCommand(toolInput.command)
        case "read_file":
            return this.readFile(toolInput.path)
        case "list_files":
            return this.listFiles(toolInput.path)
        // ...その他のツール
    }
}
```

2. 権限チェックと承認:
```typescript
// src/core/Cline.ts
async executeCommand(command: string): Promise<string> {
    const { response } = await this.ask("command", command)
    if (response === "yesButtonTapped") {
        return this.runCommand(command)
    }
    return "Command execution cancelled by user"
}
```

3. 結果のフィードバック:
```typescript
// src/core/Cline.ts
async attemptCompletion(result: string): Promise<string> {
    const { response, text } = await this.ask("completion_result", result)
    if (response === "yesButtonTapped") {
        return ""
    }
    return `User feedback: ${text}`
}
```

### 2.3 エラー処理とリカバリー

1. エラーハンドリング:
```typescript
// src/core/Cline.ts
async handleApiError(error: any): Promise<void> {
    await this.say("error", `API request failed: ${error.message}`)
    if (this.shouldRetryRequest(error)) {
        await this.retryApiRequest()
    }
}
```

2. チェックポイント管理:
```typescript
// src/core/Cline.ts
private async createCheckpoint(): Promise<void> {
    this.checkpoints.push({
        messages: [...this.clineMessages],
        apiConversationHistory: [...this.apiConversationHistory],
        apiRequestIndex: this.apiRequestIndex,
    })
}
```

3. メッセージ同期:
```typescript
// src/core/Cline.ts
async handleApiResponse(chunk: ApiResponseChunk, isFirstMessage: boolean): Promise<void> {
    if (chunk.type === "message_delta") {
        await this.handleMessageDelta(chunk, isFirstMessage)
    } else if (chunk.type === "error") {
        await this.handleApiError(chunk.error)
    }
}
```

## 3. プロンプトシステム

### 3.1 システムプロンプトの構造

1. エージェント定義:
```typescript
// src/core/prompts/system.ts
export const SYSTEM_PROMPT = async (
    cwd: string,
    supportsComputerUse: boolean,
) => `You are Cline, a highly skilled software engineer with the following capabilities:

1. Code Analysis and Development
- Read and understand source code
- Write and modify code files
- Execute system commands
- Manage project dependencies

2. Project Management
- Navigate file systems
- Handle version control
- Manage build processes
- Execute tests

3. Problem Solving
- Debug issues
- Optimize performance
- Implement new features
- Refactor code

Available tools:
${tools.map(tool => `- ${tool.name}: ${tool.description}`).join('\n')}

Current Working Directory: ${cwd}
Default Shell: ${process.platform === 'win32' ? 'powershell' : 'bash'}
`
```

### 3.2 プロンプト生成フロー

1. コンテキスト収集:
- アクティブエディタの内容
- 現在のディレクトリ構造
- 環境情報

2. プロンプト構築:
- タスク定義
- 利用可能なツール
- 環境コンテキスト

3. メッセージ履歴管理:
- 会話履歴の保持
- コンテキストウィンドウの制御
- チェックポイントの作成

## 4. システム構成

### 4.1 主要コンポーネント

1. コアエンジン（src/core/Cline.ts）:
- タスク管理
- ツール実行制御
- エラー処理
- チェックポイント管理

2. APIプロバイダー（src/api/providers/）:
```typescript
// src/api/providers/anthropic.ts
export class AnthropicProvider implements ApiProvider {
    async createMessage(
        systemPrompt: string,
        messages: Message[],
    ): Promise<ApiResponseStream> {
        // Claude 3.5 Sonnetとの通信処理
    }
}
```

3. メッセージ処理（src/core/assistant-message/）:
```typescript
// src/core/assistant-message/index.ts
export type ToolUseName =
    | "execute_command"
    | "read_file"
    | "write_to_file"
    | "replace_in_file"
    | "search_files"
    | "browser_action"
    | "use_mcp_tool"
    | "ask_followup_question"
    | "attempt_completion"
```

### 4.2 実行フローの例

1. ユーザーがタスクを入力:
```
"新しいReactコンポーネントを作成して"
```

2. システムプロンプトの生成:
```typescript
`You are Cline, a highly skilled software engineer...
Task: 新しいReactコンポーネントを作成して
Available tools: [tool descriptions]
Current directory: ${cwd}
...`
```

3. LLMの応答と処理:
```typescript
// ツールの選択と実行
{
    tool: "read_file",
    path: "package.json"  // プロジェクト設定の確認
}

{
    tool: "write_to_file",
    path: "src/components/NewComponent.tsx",
    content: "..."  // 新しいコンポーネントの作成
}
```

4. ユーザーへのフィードバック:
```typescript
await this.attemptCompletion(
    "新しいReactコンポーネントを作成しました。\n" +
    "src/components/NewComponent.tsxを確認してください。"
)
```

### 4.3 プロンプトシステムの詳細

1. プロンプトの構造:
```typescript
// src/core/prompts/system.ts
const basePrompt = `You are Cline, a highly skilled software engineer...

Your capabilities:
1. Code Analysis and Development
2. Project Management
3. Problem Solving

Available tools:
${tools.map(tool => `- ${tool.name}: ${tool.description}`).join('\n')}
`

const contextPrompt = `
Current Working Directory: ${cwd}
Environment: ${process.platform}
Active Files: ${activeFiles.join(', ')}
`

const toolUsagePrompt = `
Tool Usage Guidelines:
1. Always explain your intentions before using a tool
2. Verify file paths before operations
3. Request user confirmation for critical actions
`
```

2. プロンプトの生成フロー:
```typescript
// src/core/Cline.ts
async generatePrompt(task: string): Promise<string> {
    const context = await this.gatherContext()
    const systemPrompt = await SYSTEM_PROMPT(
        context.cwd,
        this.api.getModel().info.supportsComputerUse
    )
    return `${systemPrompt}

Task Description:
${task}

Current Context:
${context.toString()}`
}
```

3. タスク実行の例:

ユーザー入力:
```
"package.jsonのdependenciesを最新バージョンに更新して"
```

生成されるプロンプト:
```
You are Cline, a highly skilled software engineer...

Task Description:
package.jsonのdependenciesを最新バージョンに更新して

Available Tools:
- read_file: ファイルの内容を読み取り
- write_to_file: ファイルに内容を書き込み
- execute_command: システムコマンドを実行

Current Context:
Working Directory: /path/to/project
Active File: package.json
```

LLMの応答と処理:
```typescript
// 1. package.jsonの読み取り
{
    tool: "read_file",
    path: "package.json"
}

// 2. 依存関係の更新コマンド実行
{
    tool: "execute_command",
    command: "npm outdated"
}

// 3. 更新の実行
{
    tool: "execute_command",
    command: "npm update"
}
```

## 5. Roo-Codeとの比較分析

### 5.1 主要な機能追加

1. カスタムモードシステム（src/shared/modes.ts）:
```typescript
export type ModeConfig = {
    slug: string
    name: string
    roleDefinition: string
    customInstructions?: string
    groups: readonly GroupEntry[]
}
```

- 3つの基本モード:
  - **Code**: 通常のコーディングアシスタント
  - **Architect**: システム設計とアーキテクチャ分析に特化
  - **Ask**: 質問応答とドキュメント作成に特化

- モードごとの機能制限:
  - ファイルアクセス制限（例：ArchitectモードはMarkdownファイルのみ編集可能）
  - ツールグループの制限
  - カスタムプロンプトとロール定義

2. ファイル制限機能:
```typescript
export function isToolAllowedForMode(
    tool: string,
    modeSlug: string,
    customModes: ModeConfig[],
    toolRequirements?: Record<string, boolean>,
    toolParams?: Record<string, any>,
    experiments?: Record<string, boolean>,
): boolean
```

- 正規表現によるファイルパターンマッチング
- モードごとの編集権限管理
- ツール使用制限の柔軟な設定

3. プロンプトカスタマイズ:
```typescript
export type CustomModePrompts = {
    [key: string]: PromptComponent | undefined
}

export type PromptComponent = {
    roleDefinition?: string
    customInstructions?: string
}
```

- モードごとのロール定義とカスタム指示
- 動的なプロンプト生成システム
- ユーザー定義のモード追加機能

### 5.2 アーキテクチャの変更点

1. モジュール構造:
- モード関連の機能を`shared/modes.ts`に集約
- ツールグループとの連携強化
- カスタムモードの状態管理

2. セキュリティ強化:
- ファイルアクセス制限の厳格化
- モードごとの権限管理
- エラーハンドリングの改善

3. 拡張性:
- カスタムモードの動的追加
- プロンプトシステムのカスタマイズ
- ツール使用制限の柔軟な設定

### 5.3 ツールシステムの改善

1. ツールグループの体系化:
```typescript
export const TOOL_GROUPS: Record<string, ToolGroupValues> = {
    read: ["read_file", "search_files", "list_files", "list_code_definition_names"],
    edit: ["write_to_file", "apply_diff", "insert_content", "search_and_replace"],
    browser: ["browser_action"],
    command: ["execute_command"],
    mcp: ["use_mcp_tool", "access_mcp_resource"],
    modes: ["switch_mode", "new_task"],
}
```

- 機能別のグループ分け
- 権限管理との連携
- モードごとのツールアクセス制御

2. 常時利用可能なツール:
```typescript
export const ALWAYS_AVAILABLE_TOOLS = [
    "ask_followup_question",
    "attempt_completion",
    "switch_mode",
    "new_task",
] as const
```

- モード非依存の基本機能
- タスク管理とモード切り替え
- フォローアップ質問機能

3. ツール表示名の国際化対応:
```typescript
export const TOOL_DISPLAY_NAMES = {
    execute_command: "run commands",
    read_file: "read files",
    write_to_file: "write files",
    // ...
}
```

- ユーザーフレンドリーな表示名
- 将来の多言語対応の基盤
- UI表示の一貫性確保

### 5.4 MCPシステムの拡張

1. MCPサーバー構造の改善:
```typescript
export type McpServer = {
    name: string
    tools?: McpTool[]
    resources?: McpResource[]
    resourceTemplates?: McpResourceTemplate[]
    disabled?: boolean
}
```

- リソーステンプレートの導入
- サーバー状態管理の追加
- 動的なツール登録機能

2. 拡張されたMCPツール:
```typescript
export type McpTool = {
    name: string
}

export type McpResource = {
    uri: string
}

export type McpResourceTemplate = {
    uriTemplate: string
}
```

- リソースURIテンプレート機能
- 動的なリソース生成
- メタデータ管理の強化

3. MCPの設定と管理:
- サーバー作成機能の制御
- タイムアウト設定の追加
- サーバー状態の監視機能

4. モードとの統合:
- すべての基本モードでMCPツールを利用可能
- モードごとのMCPツールアクセス制御
- リソースアクセスの権限管理

### 5.5 開発の方向性の違い

1. Clineの最近の変更:
- モデル対応の拡張（o3-mini、r1モデルのサポート）
- ターミナルプロファイルの改善
- プラン/アクト時のモデル切り替え機能
- ストリーミング処理の改善

2. Roo-Codeの最近の変更:
- Discord連携機能の強化
- エディタユーティリティの改善
- JSDocs追加によるドキュメント整備
- UIの使いやすさ向上（モード一覧の表示改善）

3. 主な開発方針の違い:
- Cline: 
  - コア機能の強化に注力
  - モデル対応の拡充
  - パフォーマンスとUXの改善
- Roo-Code:
  - コミュニティ機能の強化
  - ドキュメント整備
  - UI/UXの改善に重点

4. アーキテクチャ上の違い:
- Cline: モノリシックな構造を維持
- Roo-Code: 
  - モジュール化の促進
  - カスタマイズ性の向上
  - 拡張機能の追加しやすさ

### 5.6 システムプロンプトの構造化

1. モジュール化されたプロンプト構造:
```typescript
const basePrompt = `${roleDefinition}
${getSharedToolUseSection()}
${getToolDescriptionsForMode()}
${getToolUseGuidelinesSection()}
${mcpServersSection}
${getCapabilitiesSection()}
${modesSection}
${getRulesSection()}
${getSystemInfoSection()}
${getObjectiveSection()}
${addCustomInstructions()}`
```

2. セクション別の機能:
- **Role Definition**: モードごとの役割定義
- **Tool Use**: ツールの使用方法と制限
- **MCP Servers**: MCPサーバーの設定と機能
- **Capabilities**: 利用可能な機能の定義
- **Modes**: モード切り替えと機能制限
- **Rules**: 動作規則とセキュリティ制限
- **System Info**: 環境情報とコンテキスト
- **Custom Instructions**: ユーザー定義の追加指示

3. カスタマイズ機能:
- モードごとのプロンプトカスタマイズ
- グローバルカスタム指示の追加
- 言語設定の反映
- 実験的機能の制御

4. セキュリティと制御:
- ファイルアクセス制限の統合
- ツール使用権限の管理
- MCPサーバー作成の制御
- 差分適用の制御機能

### 5.7 ツールグループシステムの進化

1. 構造化されたツールグループ:
```typescript
export const TOOL_GROUPS = {
  read: ["read_file", "search_files", "list_files", "list_code_definition_names"],
  edit: ["write_to_file", "apply_diff", "insert_content", "search_and_replace"],
  browser: ["browser_action"],
  command: ["execute_command"],
  mcp: ["use_mcp_tool", "access_mcp_resource"],
  modes: ["switch_mode", "new_task"]
}
```

2. 常時利用可能なツール:
```typescript
export const ALWAYS_AVAILABLE_TOOLS = [
  "ask_followup_question",
  "attempt_completion",
  "switch_mode",
  "new_task"
]
```

3. モード別アクセス制御:
- グループベースのアクセス制御
- モードごとの利用可能ツール設定
- カスタムモードでのツール制限機能
- 実験的機能のフラグ制御

4. ツール定義の改善:
- 表示名の国際化対応
- ツールオプションの型安全性
- グループ表示名のUI統合
- MCPツールの統合

5. 主な変更点:
- ツールグループによる整理
- アクセス制御の粒度向上
- UI/UXの一貫性確保
- 拡張性を考慮した設計

### 5.8 モードシステムの拡張

1. 基本モード構成:
```typescript
export const modes = [
  {
    slug: "code",
    name: "Code",
    roleDefinition: "You are Roo, a highly skilled software engineer...",
    groups: ["read", "edit", "browser", "command", "mcp"]
  },
  {
    slug: "architect",
    name: "Architect",
    roleDefinition: "You are Roo, a software architecture expert...",
    groups: [
      "read",
      ["edit", { fileRegex: "\\.md$", description: "Markdown files only" }],
      "browser",
      "mcp"
    ]
  },
  {
    slug: "ask",
    name: "Ask",
    roleDefinition: "You are Roo, a knowledgeable technical assistant...",
    groups: [
      "read",
      ["edit", { fileRegex: "\\.md$", description: "Markdown files only" }],
      "browser",
      "mcp"
    ]
  }
]
```

2. ファイルアクセス制御:
- 正規表現によるファイル制限
- モード別のアクセス権限
- エラーハンドリングの改善
- カスタムエラーメッセージ

3. カスタムモード機能:
- モードの上書きと追加
- ロール定義のカスタマイズ
- グループ設定のカスタマイズ
- 実験的機能の制御

4. 主な改善点:
- より柔軟なモード定義
- 厳密なファイルアクセス制御
- エラーハンドリングの強化
- カスタマイズ性の向上

5. セキュリティ強化:
- ファイルパターンの厳密な検証
- モード別の権限管理
- ツールアクセスの制御
- エラートレース機能

### 5.9 MCPシステムの実装

1. MCPサーバー管理:
```typescript
export class McpHub {
  private connections: McpConnection[] = []
  private fileWatchers: Map<string, FSWatcher> = new Map()
  
  // サーバー設定の管理
  async updateServerConnections(newServers: Record<string, any>)
  // サーバーの有効/無効切り替え
  async toggleServerDisabled(serverName: string, disabled: boolean)
  // タイムアウト設定の更新
  async updateServerTimeout(serverName: string, timeout: number)
}
```

2. ツール実行システム:
- ツール実行の権限管理
- 常時許可リストの管理
- タイムアウト制御
- エラーハンドリング

3. リソース管理機能:
- リソーステンプレートの管理
- リソースの読み取り制御
- サーバー別のリソース一覧

4. 主な改善点:
- サーバー管理の柔軟性向上
- ツール実行の安全性強化
- リソース管理の効率化
- エラーハンドリングの改善

5. セキュリティ機能:
- サーバー別の権限管理
- ツールの実行制限
- リソースアクセスの制御
- 設定ファイルの保護

### 5.10 開発の方向性の違い

1. Clineの最近の開発フォーカス:
- シェル検出機能の改善（VS Code Terminal Profilesの活用）
- OpenAI/Deepseekのストリーミング処理の最適化
- モデル切り替え機能の強化
- 推論トークンの表示改善
- o3-miniモデルのサポート追加

2. Roo-Codeの最近の開発フォーカス:
- Discordインテグレーションの改善
- JSDocs形式のドキュメント整備
- エディタユーティリティの更新
- モード設定UIの改善
- セキュリティ強化（Webhookの保護）

3. 主な方向性の違い:
- Cline: 
  - コアエンジンの機能強化
  - モデル対応の拡充
  - パフォーマンス最適化
  - ユーザー体験の向上

- Roo-Code:
  - 外部連携の強化
  - ドキュメント整備
  - セキュリティ強化
  - UI/UXの改善

4. アーキテクチャ上の違い:
- Cline: モノリシックな設計を維持
- Roo-Code: モジュール化とプラグイン化を推進

5. 開発プロセスの違い:
- Cline: 機能追加とバグ修正が中心
- Roo-Code: リファクタリングと拡張性向上が中心

### 5.11 プロジェクトの方向性の違い（READMEの比較）

1. 基本的なアプローチ:
- Cline: 
  - シンプルで直接的なアプローチ
  - コアエンジンの機能強化に注力
  - 単一のモードで一貫した体験を提供

- Roo-Code:
  - カスタマイズ可能なマルチモードアプローチ
  - 拡張性とモジュール化を重視
  - ユーザー主導のカスタマイズを推進

2. 主要機能の違い:
- Cline:
  - OpenRouter上での最適化
  - シンプルな設定と使用体験
  - 直感的なインターフェース

- Roo-Code:
  - カスタムモードシステム
  - 高度なプロンプトカスタマイズ
  - モード別のAPI設定
  - コードアクション機能

3. コミュニティアプローチ:
- Cline:
  - 中央集権的な開発モデル
  - 機能リクエストベースの改善

- Roo-Code:
  - コミュニティ主導の開発
  - カスタムモードの共有促進
  - Discordコミュニティの活性化

4. ターゲットユーザー:
- Cline:
  - 一般的な開発者
  - シンプルで効率的な開発支援を求めるユーザー

- Roo-Code:
  - パワーユーザー
  - カスタマイズを重視するユーザー
  - チーム開発での利用を想定

5. 将来の展望:
- Cline:
  - コア機能の強化
  - パフォーマンスの最適化
  - 使いやすさの向上

- Roo-Code:
  - モードシステムの拡張
  - コミュニティ機能の強化
  - カスタマイズ機能の拡充

### 4.4 セキュリティと制御

1. 権限管理:
- ファイルアクセスの制限
- コマンド実行の承認
- APIキーの保護

2. エラー処理:
- API通信エラー
- ファイル操作エラー
- コマンド実行エラー

3. 状態管理:
- チェックポイント
- ロールバック機能
- 会話履歴の管理

Clineは以下の主要なコンポーネントで構成されています：

### 1.1 コアコンポーネント

**src/core/Cline.ts**から:
```typescript
export class Cline {
    async startTask(task?: string, images?: string[]): Promise<void> {
        this.clineMessages = []
        this.apiConversationHistory = []
        await this.say("text", task, images)
        this.isInitialized = true
        let imageBlocks: Anthropic.ImageBlockParam[] = formatResponse.imageBlocks(images)
        await this.initiateTaskLoop([
            {
                type: "text",
                text: `<task>\n${task}\n</task>`,
            },
            ...imageBlocks,
        ], true)
    }
}
```

このクラスがタスクの実行を制御し、以下の主要な機能を提供します：
- タスクの開始と初期化
- LLMとの対話管理
- ツールの実行制御
- チェックポイントの管理

### 1.2 メッセージ処理

**src/core/assistant-message/index.ts**で定義されているメッセージ型:
```typescript
export type ToolUseName =
    | "execute_command"
    | "read_file"
    | "write_to_file"
    | "replace_in_file"
    | "search_files"
    | "browser_action"
    | "use_mcp_tool"
    | "ask_followup_question"
    | "attempt_completion"
```

## 2. 実行フロー

### 2.1 タスク開始

1. ユーザーからタスクを受け取る
2. `startTask()`でタスクを初期化
3. `initiateTaskLoop()`でタスク実行ループを開始

### 2.2 LLMとの対話

**src/core/Cline.ts**から:
```typescript
async *attemptApiRequest(previousApiReqIndex: number): ApiStream {
    let systemPrompt = await SYSTEM_PROMPT(
        cwd,
        this.api.getModel().info.supportsComputerUse ?? false,
        mcpHub,
        this.browserSettings,
    )
    // ...
    const truncatedConversationHistory = getTruncatedMessages(
        this.apiConversationHistory,
        this.conversationHistoryDeletedRange,
    )
    let stream = this.api.createMessage(systemPrompt, truncatedConversationHistory)
}
```

LLMとの対話は以下のステップで行われます：
1. システムプロンプトの準備
2. 会話履歴の管理
3. APIリクエストの実行
4. レスポンスのストリーミング処理

### 2.3 ツールの実行

**src/core/Cline.ts**から:
```typescript
async executeCommandTool(command: string): Promise<[boolean, ToolResponse]> {
    const terminalInfo = await this.terminalManager.getOrCreateTerminal(cwd)
    terminalInfo.terminal.show()
    const process = this.terminalManager.runCommand(terminalInfo, command)
    // ...
}
```

ツールの実行フロー：
1. ツールの種類に応じた処理の選択
2. 必要に応じてユーザーの承認を取得
3. ツールの実行
4. 結果のフォーマットと返却

## 3. システムプロンプト

**src/core/prompts/system.ts**から:
```typescript
export const SYSTEM_PROMPT = async (
    cwd: string,
    supportsComputerUse: boolean,
    mcpHub: McpHub,
    browserSettings: BrowserSettings,
) => `You are Cline, a highly skilled software engineer...`
```

システムプロンプトには以下が含まれます：
- AIエージェントの役割定義
- 利用可能なツールの説明
- ツールの使用方法と制約
- 環境設定の反映

## 4. エラー処理とリカバリー

- チェックポイントによる状態管理
- エラー発生時のリトライ機能
- コンテキストウィンドウの管理
- ツール実行の承認フロー

### 5.12 プロンプトとツールシステムの実装比較

1. プロンプト管理アーキテクチャ:
- Cline:
  ```typescript
  // src/core/prompts/system.ts
  export const SYSTEM_PROMPT = async (
    cwd: string,
    supportsComputerUse: boolean,
  ) => `You are Cline, a highly skilled software engineer...`
  ```
  - 単一ファイルでのプロンプト管理
  - シンプルな構造設計
  - 直接的なプロンプト定義

- Roo-Code:
  ```typescript
  // src/core/prompts/sections/index.ts
  export const generateSystemPrompt = async (
    config: ModeConfig,
    context: PromptContext
  ) => {
    const sections = [
      getRoleDefinition(config),
      getToolUseSection(config),
      getMcpSection(context),
      getCustomInstructions(config)
    ]
    return sections.filter(Boolean).join('\n\n')
  }
  ```
  - モジュール化されたプロンプト構造
  - セクション別の管理
  - 動的なプロンプト生成

2. ツールシステムの設計:
- Cline:
  ```typescript
  // src/core/assistant-message/index.ts
  export type ToolUseName =
    | "execute_command"
    | "read_file"
    | "write_to_file"
    | "replace_in_file"
    | "search_files"
    | "browser_action"
  ```
  - フラットなツール構造
  - シンプルな権限管理
  - 直接的なツールアクセス

- Roo-Code:
  ```typescript
  // src/shared/tool-groups.ts
  export const TOOL_GROUPS = {
    read: ["read_file", "search_files", "list_files"],
    edit: ["write_to_file", "apply_diff", "insert_content"],
    browser: ["browser_action"],
    command: ["execute_command"],
    mcp: ["use_mcp_tool", "access_mcp_resource"]
  }
  ```
  - グループベースのツール管理
  - モード別の権限制御
  - 拡張性を考慮した設計

3. MCPの実装アプローチ:
- Cline:
  - 基本的なMCPサーバー管理
  - シンプルなツール実行フロー
  - 直接的なリソースアクセス

- Roo-Code:
  ```typescript
  // src/services/mcp/McpHub.ts
  export class McpHub {
    private connections: McpConnection[] = []
    private fileWatchers: Map<string, FSWatcher> = new Map()
    
    async updateServerConnections(newServers: Record<string, any>)
    async toggleServerDisabled(serverName: string, disabled: boolean)
    async updateServerTimeout(serverName: string, timeout: number)
  }
  ```
  - 高度なサーバー管理機能
  - 動的なツール登録
  - リソーステンプレート機能

4. エラー処理とセキュリティ:
- Cline:
  - 基本的なエラーハンドリング
  - シンプルな権限チェック
  - 直接的なセキュリティ制御

- Roo-Code:
  - 詳細なエラートラッキング
  - モード別の権限管理
  - 高度なセキュリティ制御
  - カスタマイズ可能な制限機能

5. 拡張性とカスタマイズ:
- Cline:
  - コアエンジンの機能重視
  - シンプルな拡張モデル
  - 直接的な機能追加

- Roo-Code:
  - プラグインベースの拡張
  - モード別のカスタマイズ
  - 動的な機能追加
  - コミュニティ主導の開発

### 5.13 最近の開発動向の比較（コミット履歴分析）

1. Clineの最近の主要な変更:
```
- VS Code Terminal Profilesを活用したシェル検出機能の改善
- OpenAI/Deepseekのストリーミング処理の最適化
- o3-miniモデルのサポート追加
- モデル切り替え機能の強化（plan/act間）
- r1モデルのサポート強化
```

主な特徴：
- コア機能の強化に焦点
- モデル対応の拡充
- パフォーマンスと信頼性の向上
- 基本機能の最適化

2. Roo-Codeの最近の主要な変更:
```
- Webhookセキュリティの強化
- Discord連携機能の改善（PRのスレッド投稿など）
- JSDocs形式のドキュメント追加
- モード一覧表示のUI改善
- エディタユーティリティの更新
```

主な特徴：
- コミュニティ機能の強化
- ドキュメンテーションの充実
- UIの使いやすさ向上
- セキュリティ強化

3. 開発アプローチの違い:
- Cline:
  - 基盤技術の強化
  - パフォーマンス最適化
  - モデル対応の拡充
  - 信頼性の向上

- Roo-Code:
  - コミュニティ重視
  - ドキュメント整備
  - UI/UX改善
  - セキュリティ強化

4. 今後の展望:
- Cline:
  - さらなるモデル対応の拡充
  - コア機能の最適化継続
  - パフォーマンス向上

- Roo-Code:
  - コミュニティ機能の拡張
  - カスタマイズ性の向上
  - ドキュメントの充実
  - セキュリティ強化の継続

5. 技術スタックの違い:
- 共通の基盤:
  - TypeScript/Node.js
  - VS Code拡張機能
  - LLMプロバイダー（OpenAI、Anthropic、Google、Mistral）
  - MCP（Model Context Protocol）

6. コア機能の実装比較:

### プロンプト生成システム
- Cline:
  ```typescript
  // src/core/prompts/system.ts
  export const SYSTEM_PROMPT = async (
    cwd: string,
    supportsComputerUse: boolean,
  ) => `You are Cline, a highly skilled software engineer...`
  ```
  - 単一のプロンプトテンプレート
  - シンプルな条件分岐
  - 直接的な文字列生成

- Roo-Code:
  ```typescript
  // src/core/prompts/sections/index.ts
  export const generateSystemPrompt = async (
    config: ModeConfig,
    context: PromptContext
  ) => {
    const sections = [
      getRoleDefinition(config),
      getToolUseSection(config),
      getMcpSection(context),
      getCustomInstructions(config)
    ]
    return sections.filter(Boolean).join('\n\n')
  }
  ```
  - モジュール化されたセクション
  - 設定に基づく動的生成
  - カスタマイズ可能な指示セット

### ツール管理システム
- Cline:
  ```typescript
  // src/core/assistant-message/index.ts
  export type ToolUseName =
    | "execute_command"
    | "read_file"
    | "write_to_file"
    | "replace_in_file"
    | "search_files"
    | "browser_action"
  ```
  - 単一の列挙型による定義
  - シンプルな権限管理
  - 直接的なツールアクセス

- Roo-Code:
  ```typescript
  // src/shared/tool-groups.ts
  export const TOOL_GROUPS = {
    read: ["read_file", "search_files", "list_files"],
    edit: ["write_to_file", "apply_diff", "insert_content"],
    browser: ["browser_action"],
    command: ["execute_command"],
    mcp: ["use_mcp_tool", "access_mcp_resource"]
  }
  ```
  - グループベースの管理
  - 階層的な権限制御
  - モード別のアクセス制御

### エラー処理とリカバリー
- Cline:
  - 基本的なエラーハンドリング
  - シンプルなリトライ機構
  - 直接的なエラー報告

- Roo-Code:
  ```typescript
  // src/services/mcp/McpHub.ts
  export class McpHub {
    private connections: McpConnection[] = []
    private fileWatchers: Map<string, FSWatcher> = new Map()
    
    async updateServerConnections(newServers: Record<string, any>)
    async toggleServerDisabled(serverName: string, disabled: boolean)
    async updateServerTimeout(serverName: string, timeout: number)
  }
  ```
  - 高度なサーバー管理
  - 詳細なエラートラッキング
  - 動的なリカバリー機構

### モード管理システム
- Cline:
  - シンプルなモード管理
  - 基本的な権限制御
  - 直接的なモード切り替え

- Roo-Code:
  ```typescript
  // src/shared/modes.ts
  export interface ModeConfig {
    name: string;
    description: string;
    allowedTools: string[];
    allowedPaths: string[];
    customInstructions?: string;
    roleDefinition?: string;
    contextWindow?: number;
  }
  ```
  - 詳細なモード設定
  - パス別のアクセス制御
  - カスタム指示の統合
  - コンテキストウィンドウの制御

### まとめ
1. 設計思想の違い:
   - Cline: シンプルさと直接的な実装を重視
   - Roo-Code: 拡張性とカスタマイズ性を重視

2. 実装アプローチ:
   - Cline: モノリシックな構造で、機能を直接的に実装
   - Roo-Code: モジュール化された構造で、柔軟な拡張を可能に

3. 機能の重点:
   - Cline: コア機能の最適化とパフォーマンス
   - Roo-Code: カスタマイズ性とコミュニティ機能

4. 開発の方向性:
   - Cline: 基本機能の強化と安定性の向上
   - Roo-Code: 機能の拡張と柔軟性の向上

### 開発環境とビルド設定の違い
1. パッケージ管理:
   - Cline:
     ```json
     // package.json
     {
       "scripts": {
         "build": "esbuild ./src/extension.ts --bundle --outfile=dist/extension.js",
         "test": "mocha -r ts-node/register 'test/**/*.test.ts'"
       }
     }
     ```
     - シンプルなビルド設定
     - 基本的なテスト構成
     - 最小限の開発依存関係

   - Roo-Code:
     ```json
     // package.json
     {
       "scripts": {
         "build": "npm-run-all clean compile bundle",
         "test": "jest --coverage",
         "lint": "eslint src --ext ts",
         "format": "prettier --write src"
       }
     }
     ```
     - 多段階ビルドプロセス
     - 包括的なテストカバレッジ
     - コード品質ツールの統合

2. VS Code拡張の設定:
   - Cline:
     - 基本的な拡張機能設定
     - 必要最小限のアクティベーションイベント
     - シンプルなコマンド構造

   - Roo-Code:
     - 詳細な拡張機能設定
     - 多様なアクティベーションイベント
     - カスタマイズ可能なコマンド構造
     - デバッグ設定の拡充

3. 開発プロセスの違い:
   - Cline:
     - 直接的な開発フロー
     - シンプルなCI/CD
     - 基本的なコード品質チェック

   - Roo-Code:
     - 厳密な開発フロー
     - 高度なCI/CD設定
     - 包括的なコード品質管理
     - コミュニティガイドラインの整備

### LLMプロバイダーの実装比較
1. モデル統合:
   - Cline:
     ```typescript
     // src/api/providers/anthropic.ts
     export class AnthropicProvider implements Provider {
       async complete(
         messages: Message[],
         options: CompletionOptions = {}
       ): Promise<string> {
         // 直接的なAPI呼び出し
         return this.client.messages.create({
           messages,
           model: options.model || "claude-3"
         })
       }
     }
     ```
     - シンプルなプロバイダー実装
     - 直接的なAPI呼び出し
     - 基本的なエラーハンドリング

   - Roo-Code:
     ```typescript
     // src/services/llm/providers/anthropic.ts
     export class AnthropicService extends BaseProvider {
       private readonly configManager: ConfigManager
       private readonly tokenizer: Tokenizer
       
       async complete(
         messages: Message[],
         context: CompletionContext
       ): Promise<CompletionResult> {
         const config = await this.configManager.getConfig()
         const tokens = this.tokenizer.countTokens(messages)
         
         // 高度なコンテキスト管理
         if (tokens > config.maxTokens) {
           return this.handleTokenLimit(messages, context)
         }
         
         return this.client.complete({
           messages,
           model: this.selectModel(context),
           temperature: context.temperature
         })
       }
     }
     ```
     - 高度なコンテキスト管理
     - トークン制限の動的処理
     - モデル選択の最適化
     - 詳細な設定管理

2. 機能の違い:
   - Cline:
     - 基本的なモデル呼び出し
     - シンプルなエラー処理
     - 固定的な設定

   - Roo-Code:
     - 動的なモデル選択
     - トークン管理の最適化
     - 高度なエラーリカバリー
     - 柔軟な設定管理

3. 拡張性:
   - Cline:
     - 直接的なプロバイダー追加
     - シンプルなインターフェース

   - Roo-Code:
     - プラグイン形式のプロバイダー
     - 高度な抽象化レイヤー
     - カスタマイズ可能な設定

### コミュニティとドキュメント
1. ドキュメント構造:
   - Cline:
     - 基本的なREADME
     - シンプルな使用方法の説明
     - 最小限の設定ガイド

   - Roo-Code:
     - 詳細なドキュメント体系
     - チュートリアルとガイドライン
     - API リファレンス
     - コントリビューションガイド

2. コミュニティ機能:
   - Cline:
     - 基本的なIssueテンプレート
     - シンプルなPRガイドライン

   - Roo-Code:
     - Discord統合
     - 詳細なIssueテンプレート
     - コントリビューターガイドライン
     - コミュニティイベント支援

3. 開発者サポート:
   - Cline:
     - 基本的なデバッグ情報
     - エラーメッセージ

   - Roo-Code:
     - 詳細なデバッグログ
     - テレメトリー機能
     - エラートラッキング
     - パフォーマンスモニタリング

### 総合評価
1. 使用シナリオ:
   - Cline:
     - シンプルな開発環境
     - 個人開発者向け
     - 軽量な実装が必要な場合

   - Roo-Code:
     - チーム開発環境
     - エンタープライズ利用
     - カスタマイズ性重視

2. メリット・デメリット:
   - Cline:
     - ✅ シンプルで理解しやすい
     - ✅ 軽量で高速
     - ❌ カスタマイズ性に制限
     - ❌ コミュニティ機能が限定的

   - Roo-Code:
     - ✅ 高度なカスタマイズ性
     - ✅ 充実したコミュニティ機能
     - ❌ 設定の複雑さ
     - ❌ リソース要件が高い

### プロジェクトの進化
1. 初期実装（Cline）:
   ```typescript
   // src/core/Cline.ts (初期バージョン)
   export class Cline {
     private readonly provider: Provider
     private readonly config: Config
     
     async execute(command: string): Promise<void> {
       const result = await this.provider.complete([
         { role: "system", content: SYSTEM_PROMPT },
         { role: "user", content: command }
       ])
       
       await this.handleResponse(result)
     }
   }
   ```
   - シンプルな実行フロー
   - 基本的なプロンプト管理
   - 直接的なレスポンス処理

2. 主要な更新:
   a. VS Code拡張機能の強化:
      - ターミナルプロファイルの追加
      - コマンドパレットの拡充
      - シンタックスハイライトの改善

   b. モデル対応の拡充:
      - Claude-3シリーズ対応
      - GPT-4 Turboサポート
      - Mistral AIモデル統合

   c. パフォーマンス最適化:
      - ストリーミング処理の改善
      - キャッシュ機構の導入
      - メモリ使用量の最適化

3. Roo-Codeへの進化:
   ```typescript
   // src/core/RooCode.ts
   export class RooCode extends BaseAgent {
     private readonly modeManager: ModeManager
     private readonly toolRegistry: ToolRegistry
     private readonly configManager: ConfigManager
     
     async execute(command: string, context: ExecutionContext): Promise<void> {
       const mode = await this.modeManager.getCurrentMode()
       const tools = await this.toolRegistry.getToolsForMode(mode)
       
       const result = await this.provider.complete({
         messages: await this.buildMessages(command, context),
         mode,
         tools,
         config: await this.configManager.getConfig()
       })
       
       await this.handleResponse(result, context)
     }
   }
   ```
   - モード管理の導入
   - ツールレジストリの実装
   - コンテキスト管理の強化
   - 設定の柔軟化

4. システムプロンプトの進化:
   a. Cline初期:
   ```
   You are Cline, a software engineer...
   You can use the following tools:
   - execute_command
   - read_file
   - write_to_file
   ```

   b. Roo-Code現在:
   ```
   You are an AI assistant with the following capabilities:
   {{role_definition}}
   
   Available tools for current mode ({{mode_name}}):
   {{tool_descriptions}}
   
   Context and limitations:
   {{context_window}}
   {{custom_instructions}}
   ```

5. 開発の方向性の違い:
   - Cline:
     - コア機能の最適化
     - パフォーマンス向上
     - 安定性の確保

   - Roo-Code:
     - 機能の拡張
     - カスタマイズ性の向上
     - コミュニティ機能の充実

- Roo-Code独自の追加機能:
  ```
  - AWS Bedrock Runtime統合（@aws-sdk/client-bedrock-runtime）
  - 環境変数管理の強化（@dotenvx/dotenvx）
  - 高度なテスト環境
    - Jest + ts-jest
    - カスタムレポーター
  - 文字列処理の強化
    - diff-match-patch
    - fastest-levenshtein
    - string-similarity
  - 一時ファイル管理（tmp）
  - サウンド機能（sound-play）
  ```

- 開発プロセスの違い:
  - Roo-Code:
    - より厳密なテスト環境
    - 高度なlint設定
    - AWS統合による拡張性
    - 多様な文字列処理ライブラリ

  - Cline:
    - シンプルなテスト構成
    - 基本的な開発ツール
    - 軽量な依存関係
