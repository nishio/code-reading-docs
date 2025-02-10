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

将来的にはDeep Researchを社内グループウェアに接続することで、以下のような価値が期待できます：

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
