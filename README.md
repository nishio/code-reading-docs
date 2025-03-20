# code-reading-docs

これは西尾泰和が研究している「大量の情報から効率的に情報を抽出し人間にとっての価値を生み出すこと」の応用例の一つとして「AIによってアシストされたコードリーディング」をデモンストレーションするリポジトリです。大量のソースコードから効率的に情報を抽出し、リポジトリの状態を把握・改善することを目指しています。

なお「大量の情報」が「大勢の人の意見」になると「ブロードリスニング」になります。一段階抽象化すると構造としては大差ないです。

## 特徴

- **履歴解析と比較**  
  GitHubのコミット履歴をもとに、リポジトリの変遷や、複数リポジトリ間の差分をドキュメント化します。

- **効率的なコードリーディング**  
  膨大なソースコードから必要な情報を迅速に抽出し、理解をサポートします。

- **自動ドキュメント生成**  
  READMEが存在しないリポジトリに対して、AIが自動でREADMEを生成したり、不足しているドキュメントを補完します。


## 現在の状況

現段階では、一般提供は行っていません。まずは知り合いを中心に試験的に利用してもらい、フィードバックを収集する実験段階です。

## 見出し
- [Polisの質問提示順序の仕組み](polis-question-ordering.md)
- [Polisのクラスタリング実装の解説](polis_clustering_explanation.md)
  - PolisはソースコードがClojureで書かれており、僕はあまり慣れていないので長いClojureのコードをざっと見て目的の箇所を見つけることができない。そのような場合でもAIが目的の箇所を説明付きで引用してくれるのでコードリーディングが捗る。
- [Polisにおけるトピックモデルに関する現在の議論](polis_topic_model_discussion_summary)
  - Issuesからの情報抽出
- [Community Notesにおける行列分解を用いた信頼度スコアリング](community_notes_matrix_factorization.md)
- [browser-useの仕組みの解説](browser_use_analysis.md)
- [ClineとRooの実行フロー解析](cline_analysis.md)
- [Computer-useのPythonスクリプト実装ガイド](computer-use-python.md)
- [OSSのDeep Research実装の分析レポート](deep-research.md)
- [Devikaの実行フロー解説](devika-flow.md)
- [Devikaエージェントの LLM 通信フロー](devika-agent-llm-interaction.md)
- [idobata-analyst システム解説](idobata-analyst.md)