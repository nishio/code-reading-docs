# Computer-useのPythonスクリプト実装ガイド

## 概要
Computer-useをブラウザインターフェースを使用せずに、直接Pythonスクリプトから制御する方法について解説します。

## 基本的な実装方法

```python
from anthropic import Anthropic
from computer_use_demo.tools import ComputerTool, BashTool, EditTool, ToolCollection
from computer_use_demo.loop import sampling_loop

# クライアントの設定
client = Anthropic(api_key="your_api_key")

# メッセージの例
messages = [
    {
        "role": "user",
        "content": [{"type": "text", "text": "ブラウザを開いてGoogle.comにアクセスしてください"}]
    }
]

# sampling_loopの実行
await sampling_loop(
    model="claude-3-5-sonnet-20241022",
    provider="anthropic",
    system_prompt_suffix="",
    messages=messages,
    output_callback=lambda x: print(f"モデル出力: {x}"),
    tool_output_callback=lambda x, y: print(f"ツール実行結果: {x}"),
    api_response_callback=lambda x, y, z: print(f"API応答: {y}"),
    api_key="your_api_key"
)
```

## sampling_loopの動作フロー

1. 初期設定フェーズ
   - モデル（claude-3-5-sonnet）の設定
   - ツールコレクション（Computer, Bash, Edit）の初期化
   - システムプロンプトの設定
   - 環境変数（WIDTH, HEIGHT）の確認

2. メッセージ処理フェーズ
   - ユーザーからのメッセージを受信
   - Claude APIにメッセージを送信
   - モデルからの応答を処理
   - コールバック関数を通じて進捗を報告

3. ツール実行フェーズ
   - モデルが要求したツールを実行
     - ComputerTool: スクリーン操作（クリック、タイプ、スクリーンショット）
     - BashTool: コマンド実行（アプリケーション起動など）
     - EditTool: ファイル編集
   - 実行結果をモデルに返送
   - スクリーンショットや操作結果を取得

4. フィードバックループ
   - ツールの実行結果を会話履歴に追加
   - モデルが結果を分析
   - 次のアクションを決定
   - 目標達成まで2-4を繰り返し

## 利用可能な操作

### ComputerTool
- キー入力
  - `key`: 特殊キーの入力
  - `type`: テキスト入力
- マウス操作
  - `mouse_move`: カーソル移動
  - `left_click`: 左クリック
  - `right_click`: 右クリック
  - `double_click`: ダブルクリック
- 画面操作
  - `screenshot`: スクリーンショット撮影
  - `cursor_position`: カーソル位置の取得

### BashTool
- コマンド実行
- アプリケーション起動
- システム操作

### EditTool
- ファイルの作成・編集
- テキスト操作

## 重要な注意点

1. 環境設定
   - WIDTH, HEIGHTの環境変数設定が必要
   - スクリーンショットは自動的にXGA/WXGA解像度にスケーリング

2. 非同期処理
   - ツールの実行は非同期（asyncio）で行われる
   - 適切なawait処理が必要

3. エラー処理
   - エラー発生時は自動的にリトライ
   - コールバック関数でエラー情報を取得可能

4. セキュリティ
   - APIキーの適切な管理
   - 権限の制御
   - センシティブな操作の制限

## 実装例

### ブラウザ操作の例
```python
messages = [
    {
        "role": "user",
        "content": [
            {
                "type": "text",
                "text": "以下の手順を実行してください：\n1. Firefoxを開く\n2. Google.comにアクセス\n3. 検索ボックスに'Python'と入力"
            }
        ]
    }
]

await sampling_loop(
    model="claude-3-5-sonnet-20241022",
    provider="anthropic",
    messages=messages,
    # ... その他の設定
)
```

### ファイル操作の例
```python
messages = [
    {
        "role": "user",
        "content": [
            {
                "type": "text",
                "text": "テキストファイルを作成し、'Hello, World!'と書き込んでください"
            }
        ]
    }
]

await sampling_loop(
    model="claude-3-5-sonnet-20241022",
    provider="anthropic",
    messages=messages,
    # ... その他の設定
)
```

## トラブルシューティング

1. スクリーンショットの問題
   - 解像度が高すぎる場合は自動的にスケーリング
   - 必要に応じて環境変数で調整

2. タイミングの問題
   - 操作間に適切な待機時間を設定
   - スクリーンショットの遅延は_screenshot_delayで調整可能

3. エラー処理
   - コールバック関数でエラー情報を確認
   - システムプロンプトで適切な対処方法を指定

## 参考情報
- [Anthropic API Documentation](https://docs.anthropic.com)
- [Computer Use Demo Repository](https://github.com/anthropics/anthropic-quickstarts/tree/main/computer-use-demo)
