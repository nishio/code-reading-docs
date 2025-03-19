# Devikaの実行フロー解説

Devikaは、ユーザーからの指示を受け取ってから実行完了までの間、以下のような流れで動作します。AIエージェントとして、各ステップで何が行われるのかを詳しく見ていきましょう。

## 実行フローの概要

1. ユーザーから指示を受け取る
2. タスクの計画を立てる（Plannerエージェント）
3. 必要な情報を収集する（Researcherエージェント）
4. コードを生成する（Coderエージェント）
5. コードを実行し、必要に応じて修正する（Runner/Patcherエージェント）

## 1. システムの初期化

### 1.1 エージェントの初期化
[src/agents/agent.py]
```python
def __init__(self, base_model: str, search_engine: str, browser: Browser = None):
    self.planner = Planner(base_model=base_model)
    self.researcher = Researcher(base_model=base_model)
    self.formatter = Formatter(base_model=base_model)
    self.coder = Coder(base_model=base_model)
    # ... その他のエージェントの初期化
```

Devikaは複数の専門エージェントで構成されており、それぞれが特定の役割を持っています。これらのエージェントは、ユーザーの指示を理解し、実行するために協力して動作します。

## 2. タスクの実行フロー（ステップバイステップ）

### 2.1 計画立案フェーズ
[src/agents/agent.py]
```python
def execute(self, prompt: str, project_name: str) -> str:
    plan = self.planner.execute(prompt, project_name)
    planner_response = self.planner.parse_response(plan)
    reply = planner_response["reply"]
    focus = planner_response["focus"]
    plans = planner_response["plans"]
```

ユーザーからの指示を受け取ると、まずPlannerエージェントが実行計画を立てます。この計画には、タスクの目的、焦点、具体的な実行ステップが含まれます。

### 2.2 研究フェーズ
```python
research = self.researcher.execute(plan, self.collected_context_keywords, project_name=project_name)
queries = research["queries"]
ask_user = research["ask_user"]

if queries and len(queries) > 0:
    search_results = self.search_queries(queries, project_name)
```

Plannerが計画を立てた後、Researcherエージェントが必要な情報を収集します。これには以下の手順が含まれます：
1. 関連するキーワードの抽出
2. 検索クエリの生成
3. Web検索の実行
4. 必要に応じてユーザーへの質問

### 2.3 コード生成フェーズ
```python
code = self.coder.execute(
    step_by_step_plan=plan,
    user_context=ask_user_prompt,
    search_results=search_results,
    project_name=project_name
)
```

Researcherが情報を収集した後、Coderエージェントが以下の情報を基にコードを生成します：
1. Plannerが作成した実行計画
2. Researcherが収集した情報
3. ユーザーからの追加コンテキスト

### 2.4 実行フェーズ
[src/agents/runner/runner.py]
```python
def execute(self, conversation: list, code_markdown: str, os_system: str, project_path: str, project_name: str) -> str:
    prompt = self.render(conversation, code_markdown, os_system)
    response = self.llm.inference(prompt, project_name)
    valid_response = self.validate_response(response)
```

Coderが生成したコードは、Runnerエージェントによって実行されます。この段階では：
1. コマンドの実行
2. 実行結果の監視
3. エラーが発生した場合の対応
4. 必要に応じたコードの修正

## 3. エラー処理とリカバリー

[src/agents/runner/runner.py]
```python
def run_code(self, commands: list, project_path: str, project_name: str, ...):
    for command in commands:
        # ... コマンド実行
        while command_failed and retries < 2:
            # エラー発生時の処理
            if action == "patch":
                code = Patcher(base_model=self.base_model).execute(
                    conversation=conversation,
                    code_markdown=code_markdown,
                    commands=commands,
                    error=command_output,
                    system_os=system_os,
                    project_name=project_name
                )
```

実行中にエラーが発生した場合、Patcherエージェントが問題の修正を試みます。このプロセスには以下が含まれます：
1. エラーの分析
2. 修正案の生成
3. コードの修正
4. 修正後のコードの再実行

## 4. 状態管理

各ステップの実行状態は`AgentState`クラスで管理され、ユーザーに進捗が報告されます：

```python
self.agent_state.set_agent_active(project_name, True)
self.agent_state.set_agent_completed(project_name, True)
```

## まとめ

Devikaは、複数の専門エージェントが協調して動作する分散システムとして設計されています。実行フローを時系列で見ると：

1. ユーザーが指示を入力
2. Plannerが計画を立案
3. Researcherが情報を収集
4. Coderがコードを生成
5. Runnerが実行を管理
6. （必要な場合）Patcherがエラーを修正

各エージェントは独自の役割を持ち、それぞれが連携してユーザーの目的を達成します。この設計により、複雑なソフトウェア開発タスクを効率的に実行することが可能になっています。