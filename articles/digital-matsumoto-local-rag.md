---
title: "AIに自分の資料を読ませてみる――ローカルPCで作る、はじめてのRAG"
emoji: "🧠"
type: "tech"
topics: ["rag", "ollama", "python", "localllm"]
published: true
---

## はじめに：自分の資料を読んで答えるAIは、どう動いている？

創業者や経営者の過去の発言、理念、判断事例をAIに参照させ、従業員が迷ったときに対話形式で確認できるようにする。事業承継や理念浸透の文脈では、こうした生成AIの活用が考えられる。

もちろん、経営者本人をAIで置き換えるという話ではない。**従業員が理念や判断の背景へ戻るための、対話できる入口を作れないか**という発想に近い。

こうした構想を具体化すると、すぐに「どんな技術を使うのか」「どのベンダーなら作れるのか」「費用はいくらか」という話になる。だが、発注する側が仕組みをまったくイメージできないままだと、次の問題が起きる。

- 提案された構成が、自分たちの目的に合っているか判断できない
- 「人格」「知識」「会話履歴」が一緒くたになり、要件がずれる
- 見積もりのどこに費用がかかっているか分からない
- 完成後に「思っていたものと違う」と気づく
- 特定ベンダーからデータやモデルを移せるのか確認できない

それなら、本格発注の前に、Pythonやクラウドを学ぶ入口として**自分たちで最小版を一度動かした方がよい**。完成品を内製するためではなく、何が必要で、どこが難しく、何をベンダーへ確認すべきかを体で理解するためだ。

[Digital MATSUMOTO](https://github.com/m07takash/DigitalMATSUMOTO) はApache License 2.0でソースコードが公開され、個人環境でも動く構成が意識されている。これを巨大な完成品として眺めるだけでなく、まず中心にある考え方を小さく切り出してみよう、というのがこの記事の狙いである。

自分のメモ、職務経歴、読書記録をAIに読ませて相談相手にしたい。でも、いきなり「ベクトルDB」「エージェント」「ファインチューニング」と言われると、急に遠い話に見えてしまう。

そこでこの記事では、Digital MATSUMOTOの思想を入り口に、**自分のPCの中だけで動く最小のRAG**を作る。APIキーもフレームワークも使わない。Python標準ライブラリと、ローカルでモデルを動かすOllamaだけで十分だ。

作るのは、「フォルダ内のMarkdownメモを検索し、見つけた箇所を根拠付きでローカルLLMに渡して回答させる」小さなプログラムである。

:::message
この記事の目的は、高性能なチャットアプリを完成させることではない。**RAGの中で何が起きているかを、自分のファイルと出力で確認すること**だ。
:::

この記事を終えたとき、次の状態を目指す。

- 自分のPCだけで、サンプルメモへの質問と回答を動かせる
- 同じ質問への「RAGなし」と「RAGあり」の回答を比較できる
- LLMへ渡された検索結果を、自分の目で確認できる
- RAGとファインチューニングの違いを説明できる
- 本番化するときに必要な要素を洗い出せる
- ベンダーへ具体的な質問ができる

## この記事で作るもの

```text
data/*.md（自分のメモ）
        │
        │ ① 短い断片に分ける
        ▼
   chunks（断片）
        │
        │ ② embeddingモデルで数値の列にする
        ▼
  vectors（意味の座標）
        │                         質問
        │                          │
        └──── ③ 近い断片を検索 ◀──┘
                         │
                         ▼
          ④ 断片 + 質問をLLMへ渡す
                         │
                         ▼
              回答 + 参照した断片
```

これがRAG（Retrieval-Augmented Generation、検索で補強した生成）の最小形だ。LLMそのものに自分の情報を再学習させるのではない。**質問のたびに、関係しそうな資料だけを探してプロンプトに添える**。

### 先に覚える言葉は6つだけ

初めて見る言葉が続くと、処理自体は単純でも難しく見える。この記事では、次の意味で読めば十分だ。

| 言葉 | この記事での意味 | たとえるなら |
| --- | --- | --- |
| LLM | 文章を読んで回答を作るモデル | 資料を渡すと文章を書いてくれる人 |
| RAG | 質問に近い資料を探してからLLMへ渡す仕組み | 司書が本を探し、回答者の机に置く |
| チャンク | 長い資料を分けた短い断片 | 本に貼った付箋単位 |
| 埋め込み・ベクトル | 文章の意味を比較するための数字の列 | 文章を置く「意味の地図」の座標 |
| プロンプト | LLMへ渡す指示・参考資料・質問のセット | 回答者へ渡す依頼書 |
| API | プログラム同士が決まった形式で会話する窓口 | PythonからOllamaへ注文を出す受付 |

:::message
「ベクトルの数字を人間が読む」必要はない。この記事では、**文章を数字にすると意味の近さを計算できる**と理解できれば先へ進める。
:::

## Digital MATSUMOTOから何を学ぶか

[Digital MATSUMOTO](https://www.digitalmatsumoto.com/ja/digital-matsumoto/) は、人間の知識をAIが再現できるかを主題にし、特定LLMへの依存を避けつつコンテキストデザインを重視するプロジェクトだ。公開リポジトリでは、人格・知識（RAG）・会話履歴・状況・入力情報を別々のコンテキストとして扱い、必要なものを組み立ててLLMへ渡す構成が示されている。[公開リポジトリ](https://github.com/m07takash/DigitalMATSUMOTO)

全体を雑に図にすると、こうなる。

```text
Digital MATSUMOTO の全体像

データ／ナレッジ層       メモ、RAG、会話履歴、ユーザーメモリ
          │
コンテキストデザイン層   人格 + 関連知識 + 会話 + 状況を選んで整形
          │
アプリケーション層       実行制御、分析、UI
          │
インフラ層               LLM接続、埋め込み、永続化、認証
```

今回作るのは、このうち太字にした「**知識を検索し、コンテキストに入れる**」部分だけだ。

```text
今回の最小版

Markdownメモ → チャンク → 埋め込み → 類似検索 → LLMへのプロンプト
```

ここを小さく切り出すと、RAGは意外なほど単純だと分かる。一方で、実用システムが人格、会話履歴、権限、鮮度、評価まで扱う理由も見えてくる。

:::message alert
この記事のサンプルはDigital MATSUMOTOそのものを再実装するものではない。元プロジェクトのRAG、会話メモリ、複数LLM、UI、保存・分析のうち、理解に必要な最小のRAG経路だけを教材として取り出している。
:::

なお、Apache License 2.0は「何をしてもよい」という意味ではない。Digital MATSUMOTO本体のコードを複製・改変・配布する場合は、リポジトリの[LICENSE](https://github.com/m07takash/DigitalMATSUMOTO/blob/main/LICENSE)を確認し、必要な著作権・ライセンス表示などを守る必要がある。この記事のサンプルコードは、仕組みを説明するためにゼロから書いた別実装である。

## RAGと「学習」の違い

混同しやすいので、先に分けておく。

| やること | RAG | ファインチューニング |
| --- | --- | --- |
| 自分のメモの扱い | 質問のたびに検索して渡す | 学習データとしてモデルの重みに反映する |
| メモを直したとき | 検索対象を更新すればよい | 原則として再学習が必要 |
| 根拠を見せる | 参照した断片を表示しやすい | 出力がどの学習データに由来するか追いにくい |
| 向いている用途 | 頻繁に変わる個人メモ、社内文書、規程 | 文体・出力形式・振る舞いを安定して変える |

RAGは「記憶を埋め込む」というより、**必要なときに資料棚から本を出して机に置く**仕組みだ。だから古いメモを消す、誤りを直す、出典を確かめる、といった普通の情報管理がそのまま効く。

## 用意するもの

- macOS、Windows、Linuxのいずれか
- Python 3.9以上
- [Ollama](https://ollama.com/download)（ローカルでモデルを実行するためのアプリ）
- 空き容量の目安として約5GB以上

この記事では、埋め込み用に `embeddinggemma`（約622MB）、回答用に `gemma3:4b`（約3.3GB）を使う。前者は文章を意味ベクトルに変換する専門モデル、後者は文章を回答として生成するモデルだ。容量や性能の最新情報は、[embeddinggemma](https://ollama.com/library/embeddinggemma) と [Gemma 3](https://ollama.com/library/gemma3) のモデルページで確認してほしい。

メモリに余裕がないPCでは、回答用モデルを `gemma3:1b` に変えても動く。ただし、日本語での回答品質は下がりやすい。仕組みを体験する最初の一回は、小さいモデルでも問題ない。

:::message
OllamaのローカルAPIは通常 `http://localhost:11434` で動く。この記事のコードはそこだけに接続するため、質問やメモを外部のLLM APIへ送らない。ただし、モデル本体の初回ダウンロードにはネット接続が必要であり、PCのバックアップや同期先の安全性は別途考える必要がある。
:::

## Step 1：Ollamaと2つのモデルを準備する

1. [Ollamaのダウンロードページ](https://ollama.com/download) からアプリをインストールして起動する。macOSでは、公式ドキュメントのとおりアプリをApplicationsへ入れる方法が標準である。
2. ターミナルを開き、次を1行ずつ実行する。ターミナルは、文字でPCへ命令するアプリである。macOSならSpotlightで「ターミナル」と検索すれば開ける。

```bash
# 検索用モデルをPCへダウンロードする
ollama pull embeddinggemma

# 回答用モデルをPCへダウンロードする
ollama pull gemma3:4b

# ダウンロード済みモデルの一覧を表示する
ollama list
```

`pull` は「インターネットからモデルを取得する」、`list` は「PCにあるモデルを確認する」という意味だ。`ollama list` に `embeddinggemma` と `gemma3:4b` が表示されれば準備完了である。

Ollamaの埋め込みAPIは、入力したテキスト（複数可）を埋め込みベクトルへ変換して返す。検索するメモと質問は、**同じ埋め込みモデル**で数字にするのが基本である。違う地図で作った座標同士は、正しく距離を比べられないからだ。[OllamaのEmbeddingドキュメント](https://docs.ollama.com/capabilities/embeddings)

## Step 2：作業フォルダを作る

好きな場所に `local-rag` フォルダを作り、VS Codeなどのテキストエディタで開く。その中を次の構成にする。以降のコードとサンプルデータを、そのまま同じ名前で保存すればよい。

```text
local-rag/
├── rag.py             # RAG本体
└── data/
    └── profile.md     # AIに参照させたいメモ
```

図の `#` より右側は説明であり、ファイル名には含めない。`.py` はPythonプログラム、`.md` は見出しや箇条書きを書けるMarkdown文書の拡張子だ。

ターミナルで作る場合は、次でもよい。

```bash
# local-ragフォルダを作り、その中へ移動する
mkdir local-rag
cd local-rag

# メモを置くdataフォルダを作る
mkdir data
```

`rag.py` と `data/profile.md` は、エディタで新規ファイルとして作成する。

まずは練習用のメモを `data/profile.md` として保存する。実名、住所、パスワード、顧客情報などは使わない。最初は架空の内容か、公開しても困らない自分のメモだけを使おう。

```markdown
# 私についてのメモ

## 仕事
- データを使って、現場の意思決定を助ける仕事に関心がある。
- 新しい仕組みは、便利さだけでなく「現場の人が説明できるか」を大切にしたい。
- 一人で完成させるより、早い段階で利用者に見せて改善する進め方が好き。

## 学び方
- 新しい技術は、小さなサンプルを自分で動かすと理解が進む。
- 難しい言葉だけで説明されるより、入出力を見ながら学ぶ方が定着する。

## 最近のテーマ
- ローカルLLMとRAGを使い、個人メモを検索できるようにしたい。
```

この1ファイルだけでもRAGは動く。慣れたら `data/reading.md` や `data/work.md` を追加すればよい。

## Step 3：RAG本体を書く

次のコードを `rag.py` として保存する。外部Pythonパッケージは使っていない。Pythonに最初から入っている `urllib` でOllamaのローカルAPIを呼び、検索用ベクトルはプログラムのメモリ上だけに保持する。

全部を理解してから実行する必要はない。先に、コードの地図だけ確認しておこう。

| 関数 | 役割 | RAGの段階 |
| --- | --- | --- |
| `load_chunks()` / `split_text()` | Markdownメモを読み、短く分ける | ① 読む・分ける |
| `embed()` | メモと質問を数字の列へ変える | ② 数値化する |
| `cosine_similarity()` / `retrieve()` | 質問に近いメモを選ぶ | ③ 探す |
| `generate_answer()` | 選んだメモと質問から回答を作る | ④ 答える |
| `main()` | 上の処理を順番に呼び出す | 全体の進行役 |

:::message
コード内で `#` から始まる行は、人間向けのコメントであり実行されない。`def` から始まるまとまりは「関数」と呼ばれ、何度でも呼び出せる処理の部品である。`List[str]` や `Dict[str, Any]` は値の種類を示す型ヒントなので、最初は読み飛ばしても動作の理解には影響しない。
:::

```python
#!/usr/bin/env python3
"""Markdownフォルダを対象にした、理解用の最小RAG。"""

from __future__ import annotations

# コマンドライン引数（--compare など）を受け取るための標準ライブラリ。
import argparse
# Pythonの辞書と、APIで使うJSONを相互変換する。
import json
# ベクトルの長さを計算するとき、平方根を使う。
import math
# 空行などの文字パターンで文章を分割する。
import re
# Windows / macOS / Linuxの違いを吸収してファイルを扱う。
from pathlib import Path
# 型ヒント。プログラムの動作ではなく、値の種類を読みやすくする注釈。
from typing import Any, Dict, List, Tuple
# Ollamaへ接続できなかった場合のエラーと、HTTP通信に使う部品。
from urllib.error import URLError
from urllib.request import Request, urlopen

# OllamaがこのPC内で待ち受けるAPIの場所。localhostは「自分のPC」という意味。
BASE_URL = "http://localhost:11434/api"
# 検索用モデル。文章を「意味の座標（ベクトル）」へ変換する。
EMBED_MODEL = "embeddinggemma"
# 回答用モデル。検索で見つけたメモを読んで、日本語の回答を作る。
CHAT_MODEL = "gemma3:4b"
# rag.pyと同じ場所にあるdataフォルダを、メモの置き場所にする。
DATA_DIR = Path(__file__).parent / "data"
# 質問に近いメモを、上位何件までLLMへ渡すか。
TOP_K = 3


def post_ollama(endpoint: str, payload: Dict[str, Any]) -> Dict[str, Any]:
    """OllamaのローカルHTTP APIへJSONをPOSTする。"""
    # payload（Pythonの辞書）をJSONへ変換し、Ollama宛てのリクエストを作る。
    request = Request(
        f"{BASE_URL}/{endpoint}",
        data=json.dumps(payload).encode("utf-8"),
        headers={"Content-Type": "application/json"},
        method="POST",
    )
    try:
        # 最大120秒待ち、返ってきたJSONをPythonの辞書へ戻す。
        with urlopen(request, timeout=120) as response:
            return json.loads(response.read().decode("utf-8"))
    except URLError as error:
        # 長いエラーだけを出す代わりに、初心者が先に確認する項目も表示する。
        raise SystemExit(
            "Ollamaに接続できません。アプリを起動し、"
            "`ollama pull embeddinggemma` と `ollama pull gemma3:4b` を確認してください。\n"
            f"詳細: {error}"
        )


def split_text(text: str, chunk_size: int = 500, overlap: int = 80) -> List[str]:
    """段落をなるべく保ったまま、検索しやすい長さへ分割する。"""
    # 空行を段落の境界として使う。空の段落は取り除く。
    paragraphs = [p.strip() for p in re.split(r"\n\s*\n", text) if p.strip()]
    # chunksが最終的に返す断片一覧、currentが組み立て途中の断片。
    chunks: List[str] = []
    current = ""

    for paragraph in paragraphs:
        # 1段落だけで500文字を超える場合は、単純に文字数で分ける。
        if len(paragraph) > chunk_size:
            if current:
                chunks.append(current)
                current = ""
            # 80文字ずつ重ね、分割位置で文脈が完全に切れるのを和らげる。
            for start in range(0, len(paragraph), chunk_size - overlap):
                chunks.append(paragraph[start : start + chunk_size])
            continue

        # 今の断片に次の段落を足しても500文字以内かを確認する。
        candidate = f"{current}\n\n{paragraph}".strip()
        if len(candidate) <= chunk_size:
            current = candidate
        else:
            chunks.append(current)
            # 前の末尾80文字を、新しい断片の冒頭にも残す。
            current = f"{current[-overlap:]}\n\n{paragraph}".strip()

    # ループ終了時、組み立て途中の最後の断片も忘れず追加する。
    if current:
        chunks.append(current)
    return chunks


def load_chunks() -> List[Dict[str, str]]:
    """data配下のMarkdownを読み、出典名つきのチャンク一覧にする。"""
    # *.md は「拡張子が.mdのファイルをすべて」という指定。
    files = sorted(DATA_DIR.glob("*.md"))
    if not files:
        raise SystemExit(f"{DATA_DIR} に .md ファイルがありません。")

    chunks: List[Dict[str, str]] = []
    for path in files:
        # 各ファイルを分割し、ファイル名・断片番号・本文をセットで保存する。
        for number, text in enumerate(split_text(path.read_text(encoding="utf-8")), start=1):
            chunks.append({"source": path.name, "number": str(number), "text": text})
    return chunks


def embed(texts: List[str]) -> List[List[float]]:
    """テキスト群をまとめて埋め込みベクトルへ変換する。"""
    # /api/embedへ文章を送り、文章ごとの数字の列（ベクトル）を受け取る。
    response = post_ollama("embed", {"model": EMBED_MODEL, "input": texts})
    return response["embeddings"]


def cosine_similarity(a: List[float], b: List[float]) -> float:
    """2つのベクトルの向きの近さを、-1から1で返す。"""
    # 内積。2つのベクトルが同じ方向を向くほど大きくなりやすい。
    dot = sum(x * y for x, y in zip(a, b))
    # それぞれのベクトルの長さを計算する。
    length_a = math.sqrt(sum(x * x for x in a))
    length_b = math.sqrt(sum(y * y for y in b))
    # 長さの影響を除き「向き」だけを比べる。長さ0なら0.0を返す。
    return dot / (length_a * length_b) if length_a and length_b else 0.0


def retrieve(question: str, chunks: List[Dict[str, str]]) -> List[Tuple[float, Dict[str, str]]]:
    """質問に意味が近いTOP_K件の断片を選ぶ。"""
    # ① すべてのメモ断片をベクトルにする。
    chunk_vectors = embed([chunk["text"] for chunk in chunks])
    # ② 質問も、同じ埋め込みモデルでベクトルにする。
    question_vector = embed([question])[0]
    # ③ 質問と各断片の近さを計算し、「類似度 + 断片」の組にする。
    scored = [
        (cosine_similarity(question_vector, vector), chunk)
        for chunk, vector in zip(chunks, chunk_vectors)
    ]
    # ④ 類似度が高い順に並べ、上位TOP_K件だけを返す。
    return sorted(scored, key=lambda item: item[0], reverse=True)[:TOP_K]


def generate_answer(question: str, references: List[Tuple[float, Dict[str, str]]]) -> str:
    """同じ指示文を使い、参考資料の有無だけを変えて回答させる。"""
    # 検索された断片を、[1] [2] ... の出典番号つきテキストへ整形する。
    context = (
        "\n\n".join(
            f"[{index}] 出典: {chunk['source']} #{chunk['number']}\n{chunk['text']}"
            for index, (_, chunk) in enumerate(references, start=1)
        )
        if references
        else "（参考資料なし）"
    )
    # LLMへ渡す指示文。質問だけでなく、回答ルールと参考資料も一緒に渡す。
    prompt = f"""あなたは質問に日本語で簡潔に回答するアシスタントです。

参考資料があるときは、その内容だけを根拠に回答してください。
参考資料がないときは、個人について知っているふりをせず、一般論として回答してください。
参考資料に答えがないときは、推測せず「資料にはありません」と言ってください。
参考資料を使った文末には、対応する番号を [1] の形式で付けてください。

参考資料:
{context}

質問:
{question}
"""
    # /api/generateへ指示文を送り、回答を生成する。
    response = post_ollama(
        "generate",
        # stream=Falseは、回答を分割せず完成後にまとめて受け取る設定。
        # temperature=0は、回答のランダムさを抑えて比較しやすくする設定。
        {"model": CHAT_MODEL, "prompt": prompt, "stream": False, "options": {"temperature": 0}},
    )
    # APIの返答から回答本文だけを取り出し、前後の余分な空白を除く。
    return response["response"].strip()


def print_references(references: List[Tuple[float, Dict[str, str]]]) -> None:
    """LLMへ渡す検索結果を、人が確認できる形で表示する。"""
    print("--- 検索結果（この内容だけをLLMへ渡す） ---")
    for index, (score, chunk) in enumerate(references, start=1):
        # 類似度は「検索の並び順」の手掛かり。回答の正しさを表す確率ではない。
        print(f"[{index}] {chunk['source']} #{chunk['number']} / 類似度 {score:.3f}")
        print(chunk["text"])
        print()


def run_self_test() -> None:
    """Ollamaなしで、分割と類似度の最低限を確認する。"""
    # assertは、右側の条件が成立しなければエラーにする命令。
    assert len(split_text("A\n\nB", chunk_size=10, overlap=2)) == 1
    assert cosine_similarity([1.0, 0.0], [1.0, 0.0]) == 1.0
    assert cosine_similarity([1.0, 0.0], [0.0, 1.0]) == 0.0
    print("self-test: OK")


def main() -> None:
    # ① ターミナルで指定された質問やオプションを読み取る。
    parser = argparse.ArgumentParser(description="ローカル最小RAG")
    parser.add_argument("question", nargs="*", help="質問（省略時は対話入力）")
    parser.add_argument("--compare", action="store_true", help="同じ質問へのRAGなし・ありの回答を比較する")
    parser.add_argument("--show-context", action="store_true", help="LLMへ渡す検索結果を表示する")
    parser.add_argument("--test", action="store_true", help="Ollamaを使わない自己テスト")
    args = parser.parse_args()

    # ② --testが付いていたら、Ollamaを使わず自己テストだけ行って終了する。
    if args.test:
        run_self_test()
        return

    # ③ コマンドに続けて書かれた文章を質問にする。なければ入力を待つ。
    question = " ".join(args.question) or input("質問: ").strip()
    if not question:
        raise SystemExit("質問を入力してください。")

    # ④ --compareでは、同じモデル・同じ質問で参考資料の有無だけを変える。
    if args.compare:
        print("=== RAGなし：LLMの一般知識だけ ===")
        print(generate_answer(question, []))
        references = retrieve(question, load_chunks())
        print("\n=== RAGあり：検索したメモを追加 ===")
        print_references(references)
        print("--- 回答 ---")
        print(generate_answer(question, references))
        return

    # ⑤ 通常実行では、メモを読み、質問に近い断片を検索する。
    references = retrieve(question, load_chunks())

    # --show-contextがあれば、LLMへ渡す前の検索結果も表示する。
    if args.show_context:
        print_references(references)

    # ⑥ 検索結果を参考資料として渡し、最終回答を表示する。
    print("--- 回答 ---")
    print(generate_answer(question, references))


if __name__ == "__main__":
    main()
```

## Step 4：同じ質問で「RAGなし／あり」を比べる

まずは、Ollamaなしでできる自己テストを実行する。

```bash
python3 rag.py --test
# self-test: OK
```

`python3` は「Python 3でこのファイルを実行する」という命令、`--test` は通常の質問をせず簡単な動作確認だけを行うオプションだ。Windows環境などで `python3` が見つからない場合は、`python rag.py --test` も試してほしい。

次に、質問と一緒に `--compare` を付けて実行する。この比較では、どちらも同じ `gemma3:4b` を使う。変えるのは、検索したメモをプロンプトへ追加するかどうかだけだ。

```bash
python3 rag.py --compare "仕事の進め方で大切にしていることは？"
```

回答モデルを2回動かすため、通常の質問より待ち時間は長くなる。

出力例は次のようになる。モデルの実行環境によって表現は多少変わる。

```text
=== RAGなし：LLMの一般知識だけ ===
仕事では、目標を明確にし、優先順位をつけ、周囲とコミュニケーションを取ることが大切です。

=== RAGあり：検索したメモを追加 ===
--- 検索結果（この内容だけをLLMへ渡す） ---
[1] profile.md #1 / 類似度 0.xxx
# 私についてのメモ

## 仕事
- データを使って、現場の意思決定を助ける仕事に関心がある。
...

--- 回答 ---
現場の人が説明できる仕組みを大切にし、早い段階で利用者に見せながら改善する進め方を重視しています。[1]
```

見る場所は3か所だけでよい。

1. `RAGなし`：質問だけを渡した一般的な回答
2. `検索結果`：Pythonがメモのどこを選び、LLMへ渡したか
3. `RAGあり`：検索結果を根拠にした回答と `[1]` などの出典番号

:::message
画面に出る「類似度」は、質問とメモの意味がどれくらい近いかを**並べるための点数**であり、回答が正しい確率ではない。0.8なら80%正しい、という読み方はしない。
:::

違いは、モデルが急に賢くなったからではない。

| 比較 | LLMへ渡したもの | 回答の特徴 |
| --- | --- | --- |
| RAGなし | 質問だけ | 一般論には答えられるが、この人固有の考え方は分からない |
| RAGあり | 質問 + 検索したメモ | メモにある価値観を使い、参照番号付きで具体的に答えられる |

GPT、Claude、Geminiなどの一般的なLLMも、追加の資料や検索結果を渡さなければ、手元にある非公開メモの内容は知りようがない。製品によって会話履歴やメモリなどの機能は異なるが、今回比べている本質は同じである。**LLMの一般知識だけで答える場合と、質問に必要な自分の文脈を追加して答える場合の差**だ。

ここで一番見てほしいのは、RAGありの「回答」だけでなく、その前の**検索結果**だ。LLMは `data/` フォルダ全体を魔法のように知っているわけではない。今回のプログラムでは、質問に近い最大3断片だけを選び、その内容を材料として渡している。

RAGありの回答だけを試したい場合は、従来どおり次のコマンドを使える。

```bash
python3 rag.py --show-context "仕事の進め方で大切にしていることは？"
```

## うまく動かないとき

最初につまずきやすい箇所をまとめる。エラー全文を消さずに、上から順に確認してほしい。

| 症状 | 主な原因 | 確認すること |
| --- | --- | --- |
| `python3: command not found` | Pythonが未導入、またはコマンド名が違う | `python --version` も試す。Windowsでは記事中の `python3` を `python` に読み替える場合がある |
| `ollama: command not found` | Ollama未導入、またはCLIへパスが通っていない | Ollamaアプリを一度起動する。macOSは公式手順どおりCLIリンクを作成する |
| `Ollamaに接続できません` | Ollamaアプリが停止している | アプリを起動し、別のターミナルで `ollama list` が動くか確認する |
| `model ... not found` | モデルをまだ取得していない | `ollama pull embeddinggemma` と `ollama pull gemma3:4b` を再実行する |
| `embeddinggemma` の取得・実行に失敗する | Ollamaが古い | Ollamaを最新版へ更新する。モデルページではv0.11.10以降が必要と案内されている |
| 回答が非常に遅い、PCが重い | 回答モデルがPCに対して大きい | `ollama pull gemma3:1b` を実行し、`CHAT_MODEL` を `gemma3:1b` に変える |
| `資料にはありません` ばかり返る | 検索された断片に答えがない | `--show-context` で検索結果を確認し、メモの書き方や質問を見直す |

ここでも、回答だけを眺めず、コマンドがどの段階で止まったかを切り分ける。`ollama list` が動くか、`--test` が通るか、`--show-context` に期待した文章が出るか、という順番で確認すれば原因を絞りやすい。

## コードの中で何が起きたか

### 0. PythonからOllamaへ依頼する

`post_ollama()` は、PythonとOllamaの共通窓口だ。Pythonの辞書をJSONというデータ形式へ変換し、`http://localhost:11434/api` へ送る。

この記事では、次の2種類の依頼が同じ窓口を通る。

| API | 依頼すること | 返ってくるもの |
| --- | --- | --- |
| `/api/embed` | 文章を意味の座標へ変える | ベクトル（数字の列） |
| `/api/generate` | 指示文と資料を読んで回答する | 日本語の文章 |

`localhost` は自分のPCを指すため、遠隔のWebサービスへ送るURLではない。

### 1. チャンク化：資料を小さな断片へ分ける

`split_text()` はMarkdownを約500文字ごとに分ける。文書を丸ごと1件として検索すると、「仕事」「学び方」「最近のテーマ」が混ざり、質問に対する検索精度が下がるからだ。

さらに、前の断片の末尾80文字を次の断片にも少し入れている。これを**オーバーラップ**という。段落や文の境目で意味が切れるのを和らげるための、小さな保険だ。

チャンクを小さくしすぎると前後関係を失い、大きくしすぎると無関係な文章まで一緒に選ばれる。正解の文字数はない。今回の `500 / 80` は、挙動を観察するための出発点である。

### 2. 埋め込み：文章を意味の座標へ変える

`embed()` は、文章を数百個の数字からなるベクトルへ変換する。たとえば「利用者に早く見せて改善する」と「試作品を早い段階で使ってもらう」は、文字が完全一致しなくても意味が近いので、ベクトルの向きが近づきやすい。

これはキーワード検索との大きな違いだ。ただし埋め込みは「意味を完全に理解する装置」ではない。固有名詞、数字、否定、日付、最新版の区別などは弱くなり得る。実務ではキーワード検索やメタデータ条件も組み合わせる理由がここにある。

### 3. 検索：質問と近いベクトルを選ぶ

`cosine_similarity()` は、質問のベクトルと各チャンクのベクトルの**向きの近さ**を計算している。数値が大きいほど似ている、と考えればよい。

```python
return sorted(scored, key=lambda item: item[0], reverse=True)[:TOP_K]
```

この1行が検索の本体だ。今回、ベクトルはプログラム実行のたびに作り直している。データが数十ファイル程度なら、仕組みを理解するには十分である。

### 4. 生成：検索結果を「根拠」としてLLMへ渡す

最後に、検索された断片に `[1]`、`[2]` と番号をつけて、質問と一緒に `gemma3:4b` へ送る。プロンプトでは次を明示している。

```text
- 参考資料に書かれた内容だけを根拠にする
- なければ「資料にはありません」と言う
- 根拠番号を回答に付ける
```

これはLLMを完全に正直にする魔法ではない。それでも、根拠を**回答と一緒に確認できる形へ設計する**のは、RAGで最初に入れるべき安全装置だ。

ここまでを、`main()` が次の順番で呼び出している。

```text
質問を受け取る
  → load_chunks()     メモを読む・分ける
  → retrieve()        数値化して近い断片を探す
  → print_references()検索結果を人にも見せる
  → generate_answer() 検索結果と質問から回答する
```

## Step 5：3つの実験でRAGを体感する

動いたら、次の実験をしてみよう。RAGの性質がかなりはっきり見える。

### 実験A：言い換えた質問をする

```bash
python3 rag.py --show-context "新しい仕事はどう進めるのが好き？"
```

`profile.md` に「新しい仕事」という語がなくても、「早い段階で利用者に見せて改善する」という箇所が選ばれれば、意味検索が働いている。

### 実験B：資料にない質問をする

```bash
python3 rag.py --show-context "好きな休日の過ごし方は？"
```

正しい挙動は、もっともらしい趣味を創作することではない。「資料にはありません」と答えることだ。もし断定してしまうなら、プロンプトを強める、検索結果の類似度が低すぎるときは回答しない、といった対策が必要になる。

### 実験C：検索結果をわざと悪くする

`TOP_K = 3` を `TOP_K = 1` や `TOP_K = 5` に変えて、同じ質問をする。

- 少なすぎると、必要な前提が抜ける
- 多すぎると、無関係な情報が混ざる
- 回答だけでなく、`--show-context` の中身がどう変わるかを見る

RAGの改善は「モデルを賢くする」前に、**正しい資料を、適切な量で渡せているか**を観察する仕事でもある。

## 最小版から、実用版へ進めるなら

今回のコードは、RAGを分解して見るために、あえて足りないものを残している。

| 最小版の状態 | 次に足すもの | なぜ必要か |
| --- | --- | --- |
| 実行のたびに全ファイルを埋め込み直す | ベクトルの保存・差分更新 | 文書が増えても待ち時間を抑える |
| 全文を文字数で分割 | 見出し・ページ・表を意識した分割 | 出典と意味のまとまりを保つ |
| 類似度だけで選ぶ | 日付、カテゴリ、権限などのメタデータ | 「最新版だけ」「仕事だけ」を守る |
| 参照番号だけ | ファイルへのリンク、引用箇所の表示 | 人が根拠を検証できるようにする |
| その場の質問だけ | 会話履歴、短期・長期のメモリ | 継続的な対話を扱う |
| 目視で確認 | 質問集と期待する根拠による評価 | 更新しても検索品質を測る |

この表の右側を、より広く扱っているのがDigital MATSUMOTOのようなコンテキスト設計だ。RAGは「ベクトル検索を入れたら終わり」ではなく、**何を知識として保存し、いつ、誰に、どの量だけ渡すか**を設計する仕事になる。

## 一度動かすと、ベンダーに聞く質問が変わる

ここまでの最小版だけでも、本番システムを発注するときの論点が具体的になる。

| 手元で触ったもの | 本番で決めること | ベンダーへ確認する質問 |
| --- | --- | --- |
| `data/*.md` | AIが参照する正本 | どの文書を誰が更新・承認しますか。元データは当社が持ち出せますか |
| チャンク化 | 文書の分け方 | 見出し、表、PDF、音声、日付をどう分割・保持しますか |
| 埋め込み | 検索モデルと更新 | どのモデルを使い、変更分だけ再処理できますか。モデルを交換できますか |
| 類似検索 | 取得精度と絞り込み | 部門、役職、時点、公開範囲で検索対象を制限できますか |
| `--show-context` | 説明可能性 | 回答が参照した原文を、利用者と管理者が確認できますか |
| 「資料にはありません」 | 誤回答の抑制 | 根拠が弱いときに回答を止める基準は何ですか |
| サンプル質問 | 評価 | 想定する回答や期待する根拠と、継続的に比較できますか |
| ローカルAPI | 配置とデータ境界 | 何が社内・端末内に残り、何がクラウドへ送られ、ログはいつ消えますか |
| モデル名の設定 | ベンダーロックイン | LLM、ベクトルDB、クラウドを変更するとき、どのデータを移行できますか |
| Pythonコード | 費用の内訳 | 初期構築、データ整備、モデル利用、監視、改善の費用を分けて提示できますか |

特定の人の知識や価値観を参照するAIを考えるなら、その人らしい口調だけを再現しても足りない。少なくとも次を分けて設計する必要がある。

- **事実**：経歴、会社の歴史、商品、過去の出来事
- **価値観**：何を優先し、何を避けるか
- **判断事例**：どの状況で、何を根拠に、どう決めたか
- **話し方**：語彙、口調、説明の順番
- **現在の公式見解**：昔の発言と、今も有効な方針の区別

この違いを理解すると、「資料を全部入れて、それっぽく喋らせる」という曖昧な要件から、「判断事例を出典付きで検索し、現行方針と過去の発言を区別して回答する」といった検証可能な要件へ進める。

ローカル版は完成品ではない。しかし、発注側と開発側が同じ部品を指して話すための**動く要件定義書**にはなる。

### ローカルからクラウドへ移すと増えるもの

この記事のコードは、1台のPCを1人で使う前提だ。従業員がブラウザから利用するクラウドサービスにすると、RAG以外の仕事が一気に増える。

- ユーザー認証と、役職・部門ごとのアクセス制御
- 文書のアップロード、更新、承認、削除の画面
- APIキーや個人情報の安全な保存
- 通信と保存データの暗号化
- 誰が何を質問し、どの資料が使われたかの監査ログ
- 同時利用への対応、障害監視、バックアップ
- LLM・埋め込み・ストレージの利用料管理

ローカルの数百行が、そのまま本番価格になるわけではない。見積もりで確認すべきなのは、**RAGのコア以外に、どの運用・安全・評価を含んでいるか**である。まずローカル版を動かしておくと、その差分を項目ごとに話せるようになる。

## 個人のRAGで先に決めておくこと

ローカルで動くことと、安全に扱えることは同義ではない。個人メモを入れる前に、最低限ここを決めよう。

- `data/` に何を置かないか（パスワード、認証トークン、他人の個人情報、会社の持ち出し禁止情報）
- Git管理するなら、どのフォルダを `.gitignore` に入れるか
- 検索結果を表示し、誤った根拠で答えていないかどう確認するか
- 古いメモや矛盾したメモを、どう更新・保管するか

個人用RAGの品質は、モデル名よりも元のメモの整理、鮮度、出典の見え方に大きく左右される。AIに「自分を理解させる」前に、AIが参照する資料の正本を一つに決める。この地味な作業が、後から一番効く。

## まとめ

今回作ったRAGは、次の4段階だけでできている。

1. メモを小さく分ける
2. メモと質問を同じ埋め込みモデルで数値化する
3. 質問に近いメモを選ぶ
4. 選んだメモを根拠としてLLMに渡し、回答と出典を表示する

Digital MATSUMOTOの全体はもっと広い。人格、会話履歴、状況、知識、複数LLM、保存、分析までを、コンテキストとして組み立てている。しかしその中心にある感覚は、この最小版でも体験できる。

**AIを賢く見せるのは、モデルの中にすべてを詰め込むことではない。今の問いに必要な情報を選び、検証できる形で渡すことだ。**

まずは公開してもよい数行のメモで、質問、検索結果、回答を並べて眺めてみてほしい。そこから、どのメモを増やすか、どの情報を分けるか、どこまでをAIに任せるかという、自分だけの「コンテキスト設計」が始まる。

## 次は、「本人っぽさ」の材料を集める

ここまでで、RAGが「質問に近い資料を探し、その資料を根拠としてLLMへ渡す仕組み」であることは見えてきた。

では、特定の人らしい判断を返すRAGを作りたいとき、何を資料として入れればよいのだろう。

理念や口癖だけでは、その人が**どんな場面で、何を優先し、どこで例外を置いたか**までは分からない。そこで次の記事では、本人や周囲の人から具体的な判断エピソードを聞き、まず4件だけExcelへ整理する。

[「あの人なら？」を4件から集める――意思決定RAGのExcelづくり](https://zenn.dev/hobomokha/articles/decision-rag-collect-episodes)

## 参考資料

- [Digital MATSUMOTO（公式サイト）](https://www.digitalmatsumoto.com/ja/digital-matsumoto/)
- [DigitalMATSUMOTO 公開リポジトリ](https://github.com/m07takash/DigitalMATSUMOTO)
- [Ollama: Embeddings](https://docs.ollama.com/capabilities/embeddings)
- [Ollama: Generate embeddings API](https://docs.ollama.com/api/embed)
- [Ollama: macOS](https://docs.ollama.com/macos)
