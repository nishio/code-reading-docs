# browser-useの仕組みの解説

browser-useは、AIエージェントがブラウザを操作するためのフレームワークです。コミット履歴から見る開発の流れと、主要なコンポーネントの動作を解説します。

## 開発の歴史

### 1. 初期実装（2024年10月末～11月初旬）
- 最初のコミット（2024-10-31, eeb9176）でリポジトリが作成
- 基本構造の確立：
  - ブラウザ操作の基本機能実装
  - DOM要素のクリーンアップ機能
  - 要素のクリック機能の実装

### 2. 基本機能の確立（2024年11月前半）
- LLMとの統合（2024-11-01, 28edd02）
- エージェントとブラウザの連携強化
- ゴール設定と検証機能の追加（2024-11-02, 4219966）
- 視覚モデル（Vision Model）の統合（2024-11-02, c1f7bc2）

### 3. アーキテクチャの改善（2024年11月中旬～12月）
- コントローラーとエージェントの分離（2024-11-04, b5f8e46）
- 並列処理の実装（2024-12-01, cf27d9e）
- マルチアクション実行の追加（2024-12-01, 5128b6f）

### 4. 機能拡張と安定化（2024年12月～2025年1月）
- ファイルアップロード機能の実装（2024-12-04, 05300b3）
- プロキシサポートの追加（2024-12-05, 8bc1729）
- 様々なLLMモデルのサポート追加：
  - Gemini（2025-01-07, a30d2b9）
  - Bedrock（2025-01-10, 7ee4372）
  - Deepseek（2025-01-11, d06798f）

## 2. 基本的な実行フロー

### 1.1 エージェントの初期化
最もシンプルな使用例（`examples/simple.py`）から見ていきましょう：

```python
from browser_use import Agent
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model='gpt-4o')
task = 'Find the founders of browser-use and draft them a short personalized message'
agent = Agent(task=task, llm=llm)
```

### 1.2 エージェントの実行サイクル

エージェントは以下のような循環的なステップで動作します：

1. **状態の取得と解析**：
`browser_use/agent/service.py`から、エージェントは以下のような手順で動作します：

```python
async def step(self, step_info: Optional[AgentStepInfo] = None) -> None:
    state = await self.browser_context.get_state()
    self.message_manager.add_state_message(state, self._last_result, step_info, self.use_vision)
    input_messages = self.message_manager.get_messages()
    model_output = await self.get_next_action(input_messages)
```

2. **LLMへのプロンプト発行**：
`browser_use/agent/prompts.py`に定義されているSystemPromptクラスが、LLMへの指示を構築します：

```python
def important_rules(self) -> str:
    text = """
1. RESPONSE FORMAT: You must ALWAYS respond with valid JSON in this exact format:
   {
     "current_state": {
        "page_summary": "Quick detailed summary of new information...",
        "evaluation_previous_goal": "Success|Failed|Unknown...",
        "memory": "Description of what has been done...",
        "next_goal": "What needs to be done with the next actions"
     },
     "action": [
       {
         "one_action_name": {
           // action-specific parameter
         }
       }
     ]
   }
"""
```

3. **アクションの実行**：
`browser_use/controller/service.py`のControllerクラスが、LLMの指示を実際のブラウザ操作に変換します：

```python
@self.registry.action('Click element', param_model=ClickElementAction)
async def click_element(params: ClickElementAction, browser: BrowserContext):
    session = await browser.get_session()
    state = session.cached_state
    element_node = state.selector_map[params.index]
    await browser._click_element_node(element_node)
```

## 3. LLMプロンプトとエージェントの動作

### 3.1 システムプロンプト
`browser_use/agent/prompts.py`から、エージェントの基本的な指示内容を見ていきます：

```python
class SystemPrompt:
    def get_system_message(self) -> SystemMessage:
        AGENT_PROMPT = f"""You are a precise browser automation agent that interacts with websites through structured commands. 
        Your role is to:
        1. Analyze the provided webpage elements and structure
        2. Use the given information to accomplish the ultimate task
        3. Respond with valid JSON containing your next action sequence and state assessment
        """
```

このプロンプトにより、エージェントは以下の形式でレスポンスを返します：

```json
{
  "current_state": {
    "page_summary": "現在のページの状態の要約",
    "evaluation_previous_goal": "前回のゴールの評価（Success/Failed/Unknown）",
    "memory": "これまでの行動の記録",
    "next_goal": "次のアクションで達成すべきゴール"
  },
  "action": [
    {
      "action_name": {
        "parameter": "value"
      }
    }
  ]
}
```

### 3.2 エージェントの実行フロー
`browser_use/agent/service.py`から、エージェントの実行フローを解説します：

1. **状態の取得**：
```python
async def step(self, step_info: Optional[AgentStepInfo] = None) -> None:
    # ブラウザの現在の状態を取得
    state = await self.browser_context.get_state()
    
    # 状態をメッセージ履歴に追加
    self.message_manager.add_state_message(state, self._last_result, step_info)
```

2. **LLMによるアクション決定**：
- 現在の状態とタスクに基づいて、次のアクションを決定
- JSON形式でアクション指示を生成

3. **アクションの実行**：
```python
async def execute_actions(self, actions: list[dict]) -> list[ActionResult]:
    results = []
    for action in actions:
        result = await self.controller.act(
            action,
            self.browser_context
        )
        results.append(result)
    return results
```

4. **結果の評価と次のステップへ**：
- アクション実行結果を評価
- タスク完了判定
- 必要に応じて次のステップへ継続

## 4. 主要なコンポーネント

### 4.1 Controller（`browser_use/controller/service.py`）
ブラウザ操作のアクションを管理・実行する中核コンポーネントです：

```python
class Controller:
    def _register_default_actions(self):
        @self.registry.action('Navigate to URL in the current tab', param_model=GoToUrlAction)
        async def go_to_url(params: GoToUrlAction, browser: BrowserContext):
            page = await browser.get_current_page()
            await page.goto(params.url)
```

主な機能：
- アクションの登録と管理
- ブラウザコンテキストとの連携
- エラーハンドリングとリトライ処理

### 4.2 Browser（`browser_use/browser/browser.py`）
ブラウザの制御を担当する基本クラスです：

```python
class Browser:
    async def _setup_standard_browser(self, playwright: Playwright) -> PlaywrightBrowser:
        browser = await playwright.chromium.launch(
            headless=self.config.headless,
            args=[
                '--no-sandbox',
                '--disable-blink-features=AutomationControlled',
            ]
        )
        return browser

主な機能：
- ブラウザインスタンスの管理
- ページとコンテキストの制御
- DOM要素の操作とイベント処理
- スクリーンショットとビジュアル操作
```

### 4.3 Agent（`browser_use/agent/service.py`）
AIエージェントの中核となるクラスで、タスクの実行を管理します：

```python
class Agent:
    def __init__(
        self,
        task: str,
        llm: BaseChatModel,
        browser: Browser | None = None,
        use_vision: bool = True,
        # ... その他のパラメータ
    ):
        # エージェントの初期化
        self.agent_id = str(uuid.uuid4())
        self.task = task
        self.llm = llm
        self.browser = browser or Browser()
        self.use_vision = use_vision
```

主な機能：
- タスクの管理と実行制御
- LLMとの対話管理
- ブラウザ操作の調整
- 履歴と状態の追跡

## 5. 最新の機能と統合

### 5.1 マルチアクション実行
2024年12月1日の更新で追加された機能で、複数のアクションを連続して実行できます：

```python
# browser_use/agent/service.py
async def execute_multi_actions(self, actions: list[dict]) -> list[ActionResult]:
    results = []
    for action in actions:
        result = await self.execute_actions([action])
        results.extend(result)
    return results
```

### 5.2 外部サービス統合
2025年1月に追加された主な統合機能：

1. **Discord Bot統合** (2025-01-19)
```python
# integrations/discord_api.py
class DiscordBot:
    def __init__(self, token: str, llm: BaseChatModel):
        self.agent = Agent(task="", llm=llm)
        self.token = token
```

2. **カスタムユーザーエージェント** (2025-01-19)
```python
# browser_use/browser/browser.py
async def _setup_browser_context(self, browser: PlaywrightBrowser) -> BrowserContext:
    context = await browser.new_context(
        user_agent=self.config.user_agent,
        viewport=self.config.viewport
    )
    return context
```

### 5.3 高度なブラウザ制御
- iframeとシャドウDOMのサポート (2024-11-25)
- 要素のハイライト機能
- クリップボード操作のサポート
- GIF形式での操作記録

これらの機能により、より複雑なウェブサイトの自動操作や、多様なユースケースへの対応が可能になりました。
```

### 2.3 Controller（`browser_use/controller/service.py`）
ブラウザ操作のアクションを登録・実行する役割を担います：

```python
class Controller:
    def _register_default_actions(self):
        @self.registry.action('Navigate to URL in the current tab', param_model=GoToUrlAction)
        async def go_to_url(params: GoToUrlAction, browser: BrowserContext):
            page = await browser.get_current_page()
            await page.goto(params.url)
```

## 3. 実行の流れ

1. ユーザーがタスクを指定してAgentを初期化
2. Agentは与えられたタスクをもとに、SystemPromptを使ってLLMに指示を出す
3. LLMは現在の状態を解析し、次のアクションを決定
4. ControllerがLLMの指示をブラウザ操作に変換
5. 操作結果を取得し、次のステップのための状態として使用

この循環を、タスクが完了するまで繰り返します。
