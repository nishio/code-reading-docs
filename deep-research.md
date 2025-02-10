# OSSのDeep Research実装の分析レポート

## 1. この2つのツールについて

OpenAIのLLMとFirecrawl SDKを組み合わせて、与えられたトピックについて深い調査を行うツールです。今回分析した2つの実装は以下の通りです：

### deep-research-py
- Python実装のCLIツール
- [dzhng/deep-research](https://github.com/dzhng/deep-research)のPythonポート
- 単一ユーザー向けの軽量な実装

### open-deep-research
- Next.js実装のWebアプリケーション
- マルチユーザー対応
- より洗練されたUIと拡張機能を提供

## 2. Deep Researchを非公開データに拡張するメリット

1. **知識の統合と活用**
   - インターネット上の情報と非公開データを組み合わせた包括的な分析が可能
   - 組織固有のコンテキストを考慮した深い洞察の獲得

2. **セキュリティとカスタマイズ**
   - 機密情報を外部に漏らさずに分析可能
   - 組織固有のデータ形式や特定ドメインに特化した分析の実現

3. **効率的な知識発見**
   - 既存の分析パイプラインを活用した効率的な情報抽出
   - 非公開データ間の関連性発見と新しい知見の創出

## 3. ローカルのOCR済み書籍データへの適用

OCR済み書籍データへの適用は、非公開データ活用の具体的な実装例として有望です：

1. **既存の分析パイプラインの活用**
   - 段階的な深い調査プロセス
   - 文脈を考慮した関連性の分析
   - ソース追跡と引用管理

2. **必要な修正点**
   - 検索インターフェースのローカル化
   - OCRテキストに特化した前処理
   - ローカルインデックスの構築

## 4. 他の有益な拡張可能性

Deep Researchを社内グループウェアに接続することで、以下のような価値が期待できます：

1. **ナレッジマネジメントの強化**
   - 社内文書の横断的な分析と関連付け
   - 部門間の暗黙知の共有と形式知化
   - 過去の類似案件や成功事例の効率的な発見

2. **組織知の活用**
   - 専門家の知見とベストプラクティスの共有
   - 部門を超えた知識の再利用
   - 意思決定プロセスの強化

## 5. 二つの実装の比較概要

### 共通点
1. **基本アーキテクチャ**
   - Firecrawl SDKによる検索と抽出
   - OpenAIモデルによる分析
   - 再帰的な調査プロセス

### 相違点
1. **実装プラットフォーム**
   - deep-research-py: CLIベース
   - open-deep-research: Webアプリケーション

2. **機能範囲**
   - deep-research-py: 最小限の機能セット
   - open-deep-research: 付加機能あり

## 6. プロンプトフローの解説

Deep Researchの処理フローを、実際のソースコードに基づいて解説します：

### 1. 初期クエリ生成
**ファイル**: `deep-research-py/deep_research_py/deep_research.py`
```python
async def generate_serp_queries(query: str, num_queries: int = 3):
    prompt = f"""Given the following prompt from the user, generate a list of SERP queries..."""
```
ユーザーの入力から複数の検索クエリを生成し、広範な調査を可能にします。

### 2. 検索と情報抽出
**ファイル**: `deep-research-py/deep_research_py/deep_research.py`
```python
async def process_serp_result(query: str, result: SearchResponse):
    prompt = f"Given the following contents from a SERP search for the query <query>{query}</query>..."
```
Firecrawlを使用して検索を実行し、結果から重要な情報を抽出します。

### 3. 分析と計画
**ファイル**: `open-deep-research/app/(chat)/api/chat/route.ts`
```typescript
const analyzeAndPlan = async (findings: Array<{ text: string; source: string }>) => {
    const prompt = `You are a research agent analyzing findings about: ${topic}...`;
```
収集した情報を分析し、さらなる調査が必要な領域を特定します。

### 4. 最終レポート生成
**ファイル**: `deep-research-py/deep_research_py/deep_research.py`
```python
async def write_final_report(prompt: str, learnings: List[str]):
    user_prompt = f"Given the following prompt from the user, write a final report..."
```
収集したすべての情報を統合し、包括的なレポートを生成します。

このように、両実装は以下の共通したフローで動作します：
1. ユーザーの質問を理解し、複数の調査視点を生成
2. 各視点について情報を収集・抽出
3. 収集した情報を分析し、追加調査の必要性を判断
4. すべての情報を統合して最終レポートを作成

## 7. OCR済み書籍データ対応の具体的な実装方針

deep-research-pyをOCR済み書籍データに対応させるための具体的な実装方針を説明します。

### 1. 新規モジュールの追加

#### document_source.py
OCR済み書籍データの管理と検索を担当する新しいモジュールを作成します：

```python
from typing import List, Dict, Optional
from dataclasses import dataclass
import sqlite3
from whoosh.fields import Schema, TEXT, ID
from whoosh.qparser import QueryParser

@dataclass
class DocumentMetadata:
    """OCR文書のメタデータ"""
    id: str
    title: str
    author: Optional[str]
    page_number: Optional[int]

class LocalDocumentSource:
    """Firecrawlの代替となるローカルデータソース"""
    def __init__(self, index_dir: str, db_path: str):
        self.index_dir = index_dir
        self.db_path = db_path
        self._init_storage()

    async def search(self, query: str, limit: int = 5) -> Dict[str, List[Dict]]:
        """ローカル文書の検索を実行"""
        results = []
        # Whooshを使用した全文検索の実装
        return {"data": results}

    async def extract(self, doc_id: str) -> str:
        """文書の内容を抽出"""
        # SQLiteからメタデータと本文を取得
        return content
```

### 2. 既存モジュールの修正

#### deep_research.py
メインクラスを修正してローカルデータソースに対応させます：

```python
class DeepResearch:
    """Deep Research core class"""
    def __init__(self, source=None):
        # デフォルトはFirecrawl、ローカルソースも指定可能
        self.source = source or Firecrawl(
            api_key=os.environ.get("FIRECRAWL_KEY", ""),
            api_url=os.environ.get("FIRECRAWL_BASE_URL"),
        )

    async def search(self, query: str) -> SearchResponse:
        """検索を実行（ローカルソースとFirecrawl両対応）"""
        return await self.source.search(query)
```

#### prompt.py
OCR文書特有のプロンプトを追加します：

```python
def ocr_system_prompt() -> str:
    """OCR文書用のシステムプロンプト"""
    return """OCR処理された書籍の内容を分析しています。
    以下の点に注意してください：
    - OCRエラーの可能性
    - ページ番号や文書構造
    - 引用や参考文献
    - 誤認識されやすい専門用語
    """
```

### 3. CLIの拡張

#### run.py
コマンドラインインターフェースを拡張します：

```python
@app.command()
def research(
    query: str = typer.Argument(..., help="検索クエリ"),
    source: str = typer.Option("web", help="データソース（web/local）"),
    index_dir: str = typer.Option(
        "~/.deep_research/index",
        help="インデックスディレクトリ"
    ),
):
    """実行時にデータソースを選択可能に"""
    if source == "local":
        source = LocalDocumentSource(index_dir)
    else:
        source = None  # デフォルトのFirecrawlを使用
```

### 4. 必要な依存関係の追加

pyproject.tomlに以下の依存関係を追加します：

```toml
[tool.poetry.dependencies]
whoosh = "^2.7.4"  # 全文検索エンジン
sqlite-utils = "^3.35"  # SQLite操作
python-magic = "^0.4.27"  # ファイルタイプ検出
```

### 5. データ準備ツールの追加

#### tools/index_documents.py
OCR文書のインデックス作成ツールを実装します：

```python
@app.command()
def index_documents(
    input_dir: str = typer.Argument(..., help="OCRファイルのディレクトリ"),
    format: str = typer.Option("txt", help="入力形式（txt/pdf/json）"),
):
    """OCR済み文書のインデックスを作成"""
    store = LocalDocumentSource(index_dir="~/.deep_research/index")
    
    # 文書をスキャン
    for file_path in Path(input_dir).glob(f"*.{format}"):
        # メタデータの抽出
        metadata = extract_metadata(file_path)
        # 内容の読み込みと前処理
        content = preprocess_ocr_content(file_path)
        # インデックスに追加
        store.add_document(metadata, content)
```

### 6. 実装時の注意点

1. **OCRテキストの前処理**
   - 文字化けの修正
   - 改行の正規化
   - ノイズの除去

2. **検索の最適化**
   - Whooshインデックスの適切な設計
   - メタデータの効率的な管理
   - キャッシュ戦略の実装

3. **エラー処理**
   - OCRエラーの検出と報告
   - 破損ファイルのスキップ
   - インデックス更新の整合性確保

4. **パフォーマンス考慮**
   - 大規模文書への対応
   - メモリ使用量の最適化
   - 検索速度の向上
