# idobata-analyst システム解説

## 概要
idobata-analystは、多様なソース（YouTube、X/Twitter、フォームなど）から収集したコメントデータを分析し、論点ごとの立場を特定し、それらの分析結果をレポートとして生成・可視化するシステムです。

## システムアーキテクチャ

idobata-analystは、バックエンドとフロントエンドの2つのパッケージから構成されています：

- **backend**: Node.js/TypeScriptベースのサーバーサイドロジック
- **frontend**: React/TypeScriptベースのUIコンポーネント

## 分析パイプライン

分析パイプラインは以下のステップで構成されています：

1. **コンテンツの関連性チェック**
2. **関連コンテンツの抽出**
3. **立場分析**
4. **レポート生成**
5. **視覚的レポート生成**

### 1. コンテンツの関連性チェック

`extractionService.ts`の`isRelevantToTopic`メソッドは、コメントがプロジェクトのトピックに関連しているかどうかを判断します。

- **入力**: コメントテキスト、トピック
- **出力**: 関連性の有無（boolean）
- **使用モデル**: OpenRouter API経由でAIモデルを使用
- **プロンプト**: `relevance-check.txt`テンプレートを使用

### 2. 関連コンテンツの抽出

`extractionService.ts`の`extractContent`メソッドは、関連性のあるコメントから主要な主張を抽出し、簡潔で構造化されたフォーマットに変換します。

- **入力**: 生コメントテキスト、トピック
- **出力**: 抽出された主張のテキスト
- **使用モデル**: OpenRouter API経由でAIモデル
- **プロンプト**: `content-extraction.txt`テンプレートを使用

### 3. 立場分析

`stanceAnalyzer.ts`の`analyzeStances`メソッドは、各コメントの立場を特定の論点に対して分析します。

- **入力**: プロジェクトID、質問テキスト、コメントリスト、立場オプション
- **出力**: 立場分析結果のマップ（立場ごとのコメント数と該当コメント）
- **使用モデル**: OpenRouter API経由でClaudeなどのAIモデルを使用
- **プロンプト**: `stance-analysis.txt`テンプレートを使用

### 4. レポート生成

立場レポートと全体プロジェクトレポートの2種類があります：

#### 立場レポート
`stanceReportGenerator.ts`の`generateStanceReport`メソッドが処理を行います：

- **入力**: プロジェクトID、質問ID、立場分析結果
- **出力**: 立場分析のMarkdownレポート
- **使用モデル**: OpenRouter API経由でClaudeなどのAIモデルを使用
- **プロンプト**: `stance-report.txt`テンプレートを使用

#### プロジェクトレポート
`projectReportGenerator.ts`の`generateProjectReport`メソッドが処理を行います：

- **入力**: プロジェクト情報、コメントリスト
- **出力**: プロジェクト全体のMarkdownレポート
- **使用モデル**: OpenRouter API経由でClaudeなどのAIモデルを使用
- **プロンプト**: `project-report.txt`テンプレートを使用

### 5. 視覚的レポート生成

`projectVisualReportGenerator.ts`の`generateProjectVisualReport`メソッドは、Markdownレポートを基にHTML+CSSの視覚的なレポートを生成します：

- **入力**: プロジェクト情報、コメントリスト
- **出力**: HTML+CSSインフォグラフィック
- **使用モデル**: OpenRouter API経由でanthropic/claude-3.7-sonnetモデルを使用
- **プロンプト**: カスタムプロンプト（グラフィックレコーディング風のデザイン指示）

## データ構造

### 主要なモデル

#### プロジェクト（Project）

```typescript
interface Project {
  _id: string;
  name: string;
  description: string;
  questions: Question[];
}

interface Question {
  id: string;
  text: string;
  stances: Stance[];
}

interface Stance {
  id: string;
  name: string;
}
```

#### コメント（Comment）

```typescript
interface Comment {
  _id: string;
  projectId: string;
  content: string;
  extractedContent?: string;
  isRelevant?: boolean;
  sourceType?: 'youtube' | 'x' | 'form';
  sourceUrl?: string;
  stances: CommentStance[];
}

interface CommentStance {
  questionId: string;
  stanceId: string;
  confidence: number;
}
```

#### 立場分析結果（StanceAnalysis）

```typescript
interface StanceAnalysis {
  projectId: string;
  questionId: string;
  question: string;
  stanceAnalysis: {
    [stanceId: string]: {
      count: number;
      comments: string[];
    }
  };
  analysis: string; // Markdownフォーマットのレポート
}
```

### フロントエンド表示

分析結果は以下の3つの形式で表示されます：

1. **立場分析レポート** (`StanceReport.tsx`)
   - 特定の論点に対する立場ごとの分析を表示
   - 円グラフで各立場の割合を可視化（`StanceGraphComponent.tsx`）
   - コメントサンプルとMarkdownレポートを表示

2. **プロジェクトレポート** (`ProjectReport.tsx`)
   - プロジェクト全体のMarkdownレポートを表示
   - 論点間の関連性や全体的なパターンを説明

3. **ビジュアルレポート** (`ProjectVisualReport.tsx`)
   - HTML+CSSで視覚的に強化されたグラフィカルレポート
   - グラフィックレコーディング風のデザイン

## プロンプトテンプレート

分析のために使用される主要なプロンプトテンプレート：

1. **relevance-check.txt**: コメントがトピックに関連しているかを判断
2. **content-extraction.txt**: コメントから主張を抽出して整形
3. **stance-analysis.txt**: コメントの立場を分析
4. **stance-report.txt**: 立場分析のレポートを生成
5. **project-report.txt**: プロジェクト全体のレポートを生成
6. **question-generation.txt**: コメントから論点と立場を自動生成

これらのテンプレートは`PromptTemplate`クラスによって管理され、変数の置換などの処理が行われます。

## キャッシュと再生成

分析結果はMongoDBにキャッシュされ、必要に応じて再生成することが可能です：

- 各分析メソッドは`forceRegenerate`パラメータをサポート
- 既存の分析結果がある場合はそれを使用し、ない場合や強制再生成が指定された場合は新たに生成
- 分析結果はプロジェクトIDや質問IDで索引付けされる

## まとめ

idobata-analystは、AI（主にClaudeなどのモデル）を活用して、多様なソースからのコメントを分析し、論点ごとの立場を特定・集計し、それらの分析結果を様々な形式（Markdown、グラフ、HTML+CSS視覚化）で提示するシステムです。分析プロセスは複数のステップから構成され、各ステップでは専用のプロンプトテンプレートを使用してAIにタスクを指示します。
