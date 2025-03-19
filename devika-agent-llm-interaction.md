# Devikaエージェントの LLM 通信フロー

このドキュメントでは、Devikaの各エージェント（Planner、Researcher、Coder、Runner）がLarge Language Models (LLMs)とどのように通信しているかを詳しく説明します。

## 目次

1. [システム概要](#システム概要)
2. [共通の LLM 通信基盤](#共通の-llm-通信基盤)
3. [エージェント別通信フロー](#エージェント別通信フロー)
   - [Planner エージェント](#planner-エージェント)
   - [Researcher エージェント](#researcher-エージェント)
   - [Coder エージェント](#coder-エージェント)
   - [Runner エージェント](#runner-エージェント)
4. [レスポンス検証とリトライメカニズム](#レスポンス検証とリトライメカニズム)

## システム概要

Devikaは複数のAIエージェントが協調して動作するシステムです。各エージェントは特定の役割を持ち、LLMとの通信を通じてタスクを実行します。

```
[ユーザー] → Planner → Researcher → Coder → Runner
                ↑          ↑          ↑         ↑
                └──────────┴──────────┴─────────┘
                       LLM通信基盤
```

## 共通の LLM 通信基盤

### LLMクライアントインターフェース

[src/llm/llm.py]
```python
class LLM:
    def inference(self, prompt: str, project_name: str) -> str:
        """
        LLMに対してプロンプトを送信し、応答を取得する
        """
        pass
```

### サポートされているLLMプロバイダー

1. **Claude (Anthropic)**
   [src/llm/claude_client.py]
   ```python
   class ClaudeClient(LLM):
       def __init__(self, model: str = "claude-3-opus-20240229"):
           self.client = Anthropic()
           self.model = model
   ```
   - モデル: claude-3-opus-20240229, claude-3-sonnet-20240229, claude-3-haiku-20240307
   - 特徴: 高度な推論能力、長いコンテキスト

2. **OpenAI**
   [src/llm/openai_client.py]
   ```python
   class OpenAIClient(LLM):
       def __init__(self, model: str = "gpt-4"):
           self.client = OpenAI()
           self.model = model
   ```
   - モデル: gpt-4, gpt-4-turbo
   - 特徴: 汎用的な性能、安定した出力

3. **Gemini (Google)**
   [src/llm/gemini_client.py]
   ```python
   class GeminiClient(LLM):
       def __init__(self, model: str = "gemini-pro"):
           self.client = genai.GenerativeModel(model)
   ```
   - モデル: gemini-pro
   - 特徴: 高速な推論、コスト効率

### プロンプトテンプレートエンジン

[src/agents/agent.py]
```python
def render_prompt(self, template_path: str, **kwargs) -> str:
    env = Environment(loader=FileSystemLoader(self.template_dir))
    template = env.get_template(template_path)
    return template.render(**kwargs)
```

各エージェントは、このプロンプトテンプレートエンジンを使用して：
1. テンプレートファイル（.jinja2）からプロンプトを生成
2. 動的なパラメータを注入
3. LLMクライアントの`.inference()`メソッドを呼び出し

## エージェント別通信フロー

### Planner エージェント

[src/agents/planner/planner.py]
```python
class Planner(Agent):
    def __init__(self, base_model: str):
        super().__init__(base_model)
        self.template_path = "planner/prompt.jinja2"
```

#### プロンプトテンプレート

[src/agents/planner/prompt.jinja2]
```jinja2
You are Devika, an AI Software Engineer.

The user asked: {{ prompt }}

Based on the user's request, create a step-by-step plan to accomplish the task.

Follow this format for your response:
Project Name: <プロジェクト名>
Your Reply to the Human Prompter: <ユーザーへの返信>
Current Focus: <現在の焦点>
Plan:
- [ ] Step 1: <最初のステップ>
- [ ] Step 2: <次のステップ>
...
Summary: <計画の要約>
```

#### 入力パラメータ
- `prompt`: ユーザーからの指示文（文字列）

#### プロンプトの特徴
1. AIエンジニアとしての役割を明確に定義
2. ステップバイステップの計画フォーマットを指定
3. ブラウザとサーチエンジンへのアクセスを考慮
4. シンプルなタスクは1-2ステップに抑える指示

#### レスポンス検証

[src/agents/planner/planner.py]
```python
@validate_responses
def validate_response(self, response: str) -> dict | bool:
    required_fields = ["project", "reply", "focus", "plans", "summary"]
    if not all(field in response for field in required_fields):
        return False
    return {
        "project": response["project"],
        "reply": response["reply"],
        "focus": response["focus"],
        "plans": response["plans"],
        "summary": response["summary"]
    }
```

#### 重要な制約
- 人間らしい返信を要求（"As an AI"で始めない）
- 各ステップは具体的で明確なアクション
- 計画は実装からテストまでカバー
- 指定されたフォーマット以外は拒否

#### 通信フロー図

```
[ユーザー入力] → Planner
    ↓
1. プロンプトテンプレート読み込み
    ↓
2. 動的パラメータ注入
    ↓
3. LLMにプロンプト送信
    ↓
4. レスポンス検証
    ↓
[実行計画]
```

### Researcher エージェント

[src/agents/researcher/researcher.py]
```python
class Researcher(Agent):
    def __init__(self, base_model: str):
        super().__init__(base_model)
        self.template_path = "researcher/prompt.jinja2"
```

#### プロンプトテンプレート

[src/agents/researcher/prompt.jinja2]
```jinja2
For the provided step-by-step plan, write all the necessary search queries to gather information from the web that the base model doesn't already know.

Step-by-Step Plan:
{{ step_by_step_plan }}

Keywords for Search Query: {{ contextual_keywords }}

Only respond in the following JSON format:
{
    "queries": ["<QUERY 1>", "<QUERY 2>", "<QUERY 3>"],
    "ask_user": "<ASK INPUT FROM USER IF REQUIRED>"
}
```

#### 入力パラメータ
- `step_by_step_plan`: Plannerが生成した実行計画（文字列）
- `contextual_keywords`: 検索に使用する文脈キーワード（文字列）

#### プロンプトの特徴
1. 既知の情報は検索しない（モデルの学習データに含まれる基本的な情報は除外）
2. 最大3つのクエリに制限
3. コンテキストキーワードを活用した具体的な検索
4. 基本的なHow-toは検索対象外

#### レスポンス検証

[src/agents/researcher/researcher.py]
```python
@validate_responses
def validate_response(self, response: str) -> dict | bool:
    if "queries" not in response or "ask_user" not in response:
        return False
    if not isinstance(response["queries"], list):
        return False
    if len(response["queries"]) > 3:
        response["queries"] = response["queries"][:3]
    return {
        "queries": response["queries"],
        "ask_user": response["ask_user"]
    }
```

#### 重要な制約
- 基本的なセットアップやインストール方法の検索は禁止
- 高度で具体的なクエリのみを生成
- クエリが不要な場合は空配列を返す
- 1つのJSONオブジェクトのみを返す

#### 通信フロー図

```
[実行計画] → Researcher
    ↓
1. プロンプトテンプレート読み込み
    ↓
2. 計画とキーワードを注入
    ↓
3. LLMにプロンプト送信
    ↓
4. レスポンス検証
    ↓
[検索クエリ]
```

### Coder エージェント

[src/agents/coder/coder.py]
```python
class Coder(Agent):
    def __init__(self, base_model: str):
        super().__init__(base_model)
        self.template_path = "coder/prompt.jinja2"
```

#### プロンプトテンプレート

[src/agents/coder/prompt.jinja2]
<pre>
```jinja2
Project Step-by-step Plan:
```
{{ step_by_step_plan }}
```

Context From User:
```
{{ user_context }}
```

Context From Knowledge Base:
{% if not knowledge_base_context %}
No context found.
{% else %}
{% for query, result in search_results.items() %}
Query: {{ query }}
Result:
```
{{ result }}
```
---
{% endfor %}
{% endif %}

Generate code following this format:
~~~
File: `path/to/file`:
```language
code content
```
~~~
```
</pre>

#### 入力パラメータ
- `step_by_step_plan`: Plannerが生成した実行計画（文字列）
- `user_context`: ユーザーから提供された追加コンテキスト（文字列）
- `search_results`: Researcherが収集した情報（辞書型）

#### プロンプトの特徴
1. 計画、ユーザーコンテキスト、検索結果を統合
2. 段階的な思考プロセスを要求
3. 知識ベースからの情報の活用
4. クリーンで文書化されたコードの生成を指示

#### レスポンス検証

[src/agents/coder/coder.py]
```python
@validate_responses
def validate_response(self, response: str) -> dict | bool:
    try:
        files = {}
        pattern = r"File: `([^`]+)`:\n```[a-z]*\n(.*?)\n```"
        matches = re.finditer(pattern, response, re.DOTALL)
        
        for match in matches:
            filepath = match.group(1)
            content = match.group(2)
            files[filepath] = content
            
        if not files:
            return False
            
        return {"files": files}
    except Exception:
        return False
```

#### 重要な制約
- コードは最初の試行で動作する必要がある
- 最も適切なライブラリを選択
- ファイル拡張子は正確に指定
- 必要なファイル（requirements.txt, Cargo.tomlなど）も含める
- ネストされたディレクトリ構造を正確に反映
- 実装が不可能な場合はコメントで説明
- 指定されたMarkdownフォーマット以外は拒否

#### 通信フロー図

```
[実行計画 + 検索結果] → Coder
    ↓
1. プロンプトテンプレート読み込み
    ↓
2. 計画・コンテキスト・検索結果を注入
    ↓
3. LLMにプロンプト送信
    ↓
4. レスポンス検証
    ↓
[生成コード]
```

### Runner エージェント

[src/agents/runner/runner.py]
```python
class Runner(Agent):
    def __init__(self, base_model: str):
        super().__init__(base_model)
        self.template_path = "runner/prompt.jinja2"
```

#### プロンプトテンプレート

[src/agents/runner/prompt.jinja2]
<pre>
```jinja2
You are Devika, an AI Software Engineer. You have been talking to the user and this is the exchange so far:

```
{% for message in conversation %}
{{ message }}
{% endfor %}
```

Full Code:
~~~
{{ code_markdown }}
~~~

User's last message: {{ conversation[-1] }}
System Operating System: {{ system_os }}

Generate commands in this JSON format:
{
    "commands": [
        "command1",
        "command2",
        ...
    ]
}
```
</pre>

#### 入力パラメータ
- `conversation`: ユーザーとの会話履歴（リスト）
- `code_markdown`: 生成されたコードのMarkdown形式（文字列）
- `system_os`: 実行環境のOS情報（文字列）

#### プロンプトの特徴
1. 会話コンテキストの維持
2. コード全体の把握
3. OS固有の実行コマンドの生成
4. プロジェクトディレクトリを基準とした実行

#### レスポンス検証

[src/agents/runner/runner.py]
```python
@validate_responses
def validate_response(self, response: str) -> dict | bool:
    if "commands" not in response:
        return False
    if not isinstance(response["commands"], list):
        return False
    return {
        "commands": response["commands"]
    }
```

#### 重要な制約
- コードは「自分が書いた」という前提で扱う
- プロジェクトディレクトリ内での実行を前提
- cdコマンドの使用は禁止
- OS互換性のあるコマンドを生成
- 指定されたJSON形式以外は拒否

#### 通信フロー図

```
[生成コード + 会話履歴] → Runner
    ↓
1. プロンプトテンプレート読み込み
    ↓
2. コード・会話・OS情報を注入
    ↓
3. LLMにプロンプト送信
    ↓
4. レスポンス検証
    ↓
[実行コマンド]
```

## レスポンス検証とリトライメカニズム

### validate_responses デコレータ

このデコレータは、LLMからのレスポンスを適切なJSON形式に変換します。複数の検証戦略を順番に試行します：

```python
# src/services/utils.py
@wraps(func)
def wrapper(*args, **kwargs):
    response = args[1]
    
    # 戦略1: 直接JSONとしてパース
    try:
        response = json.loads(response)
        args[1] = response
        return func(*args, **kwargs)
    except json.JSONDecodeError:
        pass

    # 戦略2: コードブロック内のJSONをパース
    try:
        pattern = r"```(?:json)?\s*([\s\S]*?)\s*```"
        match = re.search(pattern, response)
        if match:
            json_str = match.group(1)
            response = json.loads(json_str)
            args[1] = response
            return func(*args, **kwargs)
    except (json.JSONDecodeError, AttributeError):
        pass

    # 戦略3: テキスト内のJSON部分を抽出
    try:
        pattern = r"\{[\s\S]*\}"
        match = re.search(pattern, response)
        if match:
            json_str = match.group(0)
            response = json.loads(json_str)
            args[1] = response
            return func(*args, **kwargs)
    except (json.JSONDecodeError, AttributeError):
        pass

    # 戦略4: 各行を個別にJSONとしてパース
    try:
        lines = response.split("\n")
        for line in lines:
            try:
                response = json.loads(line)
                args[1] = response
                return func(*args, **kwargs)
            except json.JSONDecodeError:
                continue
    except AttributeError:
        pass
```

### エージェント固有の検証ロジック

各エージェントは、validate_responsesデコレータを使用して独自の検証ロジックを実装しています：

#### Researcher エージェント
```python
# src/agents/researcher/researcher.py
@validate_responses
def validate_response(self, response: str) -> dict | bool:
    if "queries" not in response and "ask_user" not in response:
        return False
    return {
        "queries": response["queries"],
        "ask_user": response["ask_user"]
    }
```

#### Runner エージェント
```python
# src/agents/runner/runner.py
@validate_responses
def validate_response(self, response: str) -> dict | bool:
    if "commands" not in response:
        return False
    return {
        "commands": response["commands"]
    }
```

### retry_wrapper デコレータ

エラーが発生した場合や無効なレスポンスを受け取った場合のリトライロジックを提供します：

```python
# src/services/utils.py
def retry_wrapper(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        max_retries = 5
        wait_time = 2

        for attempt in range(max_retries):
            try:
                result = func(*args, **kwargs)
                if result is not False:
                    return result
            except Exception as e:
                print(f"Error occurred: {str(e)}")
                
            if attempt < max_retries - 1:
                print(f"Retrying in {wait_time} seconds...")
                time.sleep(wait_time)
                continue
            else:
                print("All retry attempts failed")
                return False

    return wrapper
```

### リトライメカニズムの特徴

1. **段階的な検証**
   - 複数のJSONパース戦略を順番に試行
   - エージェント固有の検証ロジックを適用
   - 失敗時は次の戦略に移行

2. **エラー回復**
   - 最大5回のリトライ
   - 2秒間のウェイト時間
   - エラーメッセージの通知
   - 全リトライ失敗時の適切なエラーハンドリング

3. **柔軟な応答形式**
   - プレーンJSON
   - Markdownコードブロック内のJSON
   - テキスト中のJSON
   - 行単位のJSON

このような多層的な検証とリトライの仕組みにより、LLMからの応答を柔軟かつ堅牢に処理することが可能になっています。