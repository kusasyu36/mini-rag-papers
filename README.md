# Mini RAG for Research Papers / 小さな論文RAGプロジェクト

Google Colab 上で動く、**小規模な Retrieval-Augmented Generation (RAG)** の学習用プロジェクトです。

約30本の論文（vision-language model, multimodal AI など）を対象に、

> 自分で用意した論文コーパス  
> → ベクトル検索（Retrieval）  
> → Gemini API による要約生成（Generation）

という **RAGの一連の流れ** を、最小構成で体験することを目的としています。

---

## 1. What this project does / このプロジェクトでやっていること

自然言語の質問（日本語・英語）を入力すると、このプロジェクトは：

1. 各論文の **Title + Abstract** を `sentence-transformers` で埋め込みベクトルに変換する  
2. 埋め込みとメタデータを **Chroma** のベクターストアに保存する  
3. ユーザーの質問文を埋め込みに変換し、**意味的に近い論文 Top-k** を類似検索で取得する  
4. 取得した論文を番号付き `[文献1]..[文献k]` で並べたプロンプトを自動生成する  
5. そのプロンプトを **Gemini API（無料枠）** に渡し、  
   「これらの論文だけを根拠にした日本語サーベイ」を生成する  

つまり、**自分専用の小さな論文データベースに対するRAG** を Colab 上で完結させたミニプロジェクトです。

---

## 2. Architecture / アーキテクチャ概要

### 2.1 Data

- `data/papers.csv`  
  - 列：`id, title, year, abstract, url`  
  - 約30本の論文情報を手動で整理したものです（vision-language foundation model 周辺）

### 2.2 Embeddings (E)

- ライブラリ：`sentence-transformers`  
- モデル：`sentence-transformers/all-MiniLM-L6-v2`  
- 入力：  
  - `"Title: ...\n\nAbstract: ..."` という形式のテキスト  
- 出力：  
  - 各論文あたり 384 次元の埋め込みベクトル  

> ポイント：**意味が近い論文ほど、ベクトル空間でも近い位置に来る** ように学習されたモデルを利用しています。

### 2.3 Vector Store (R)

- ライブラリ：`chromadb`  
- 永続化パス：`vectorstore/`（Google Drive 上）  
- コレクション名：`"papers"`  

関数 `retrieve_papers(query, k)` では：

1. 質問文 `query` を埋め込みに変換  
2. Chroma に対して類似検索（k件）を実行  
3. 各ヒットについて：
   - ランク
   - 距離（小さいほど近い）
   - タイトル・年・URL
   - Title+Abstract 全文

をまとめた dict のリストとして返します。

### 2.4 Generation (G)

- ライブラリ：`google-generativeai`  
- モデル例：`gemini-2.0-flash`（free tier 向け）  

関数 `rag_answer_with_gemini(query, k)` は：

1. `retrieve_papers(query, k)` で類似論文を取得（R）  
2. 論文を `[文献1]..[文献k]` という形で並べたプロンプトを組み立て  
3. 「**以下の文献だけを根拠に日本語で答えること**」という制約付きで Gemini に送信（G）  
4. 生成された日本語サーベイと、使用した文献リストを出力  

---

## 3. Repository contents / リポジトリ構成

- `mini_rag_free.ipynb`  
  メインの Colab ノートブック。以下をすべて含みます：
  - Drive マウントとフォルダ作成
  - `papers.csv` の読み込み・前処理
  - sentence-transformers による埋め込み生成
  - Chroma コレクション作成・登録
  - `retrieve_papers` / `build_llm_prompt` / `rag_answer_with_gemini` の実装
  - Gemini API との接続（free tier）

- `data/papers.csv`  
  RAGのターゲットとなる論文リスト（小規模なサンプル）。  
  内容は自由に差し替えて、別のテーマでRAGを試すこともできます。

- `README.md`  
  このファイル。

- `requirements.txt`（作成した場合）  
  Colab以外の環境で動かすための最低限の依存関係。

---

## 4. How to run on Google Colab / 実行方法

### 4.1 ノートブックを開く

以下のいずれかの方法で `mini_rag_free.ipynb` を Colab で開きます。

- 直接URLで開く（例）  

  ```text
  https://colab.research.google.com/github/kusasyu36/mini-rag-papers/blob/main/mini_rag_free.ipynb
