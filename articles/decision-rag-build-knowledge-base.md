---
title: "Excelの判断記録をAIに読ませる――RAG投入データをPythonで作る"
emoji: "🔧"
type: "tech"
topics: ["rag", "python", "excel", "llm"]
published: false
---

> 連載「小さく作る意思決定RAG」の第2回。
> [第1回](https://zenn.dev/hobomokha/articles/decision-rag-collect-episodes)では、本人・周囲・公式記録から判断エピソードを4件集め、Excelへ整理した。今回は、そのExcelからRAGへ入れてよい行だけを取り出す。

## はじめに：Excelの1行が、どう回答へつながるのか

前回、こんな判断エピソードを作った。

```text
場面: 企画レビュー会議
状況: 若手が、完成前の企画を見せるべきか迷っていた
回答: 3割でも見せてよい。利用者の困りごとと仮説を1枚にする
理由: 完成後に方向違いへ気づく損失の方が大きい
例外: 安全や法令が関わるなら、公開範囲と確認者を先に決める
```

では、この1行がどうやって「本人なら、たぶんこう答える」という出力に変わるのか。

流れはこうだ。

```text
Excel / CSV
  │ 使ってよい判断だけ選ぶ
  ▼
RAG投入用Markdown
  │ 場面・質問・理由・例外をベクトル化
  ▼
今の質問と似たエピソードを検索
  │
  ▼
検索結果を根拠としてLLMへ渡す
  │
  ▼
新しい場面へ判断パターンを当てはめた回答
```

RAGが本人の人格を作るわけではない。**今の質問に近い過去の判断を、回答を作る瞬間だけLLMの机へ置く。** その判断に優先順位や例外が含まれていれば、一般論ではなく、その人の考え方を反映した答えに近づく。

## 今回のゴール

この記事でやることは一つだけだ。

**Excelの収集台帳から、承認済みかつ現在有効な判断だけをMarkdownへ変換し、前回作ったローカルRAGで回答の差を比べる。**

## なぜExcelをそのままRAGへ入れないのか

収集台帳には、情報提供者の名前、確認前の伝聞、非公開の補足、古くなった判断が含まれる。全部をベクトルDBへ入れる必要はない。

```text
収集台帳
  ├─ 判断内容: 状況、回答、理由、結果、例外
  ├─ 管理情報: 情報源、同意、利用範囲、確認者
  └─ 履歴: 未確認の発言、古い判断、置き換え先
             │
             │ 使ってよい行と列だけ抽出
             ▼
RAG投入データ
  └─ ID、場面、状況、問い、回答、理由、例外、タグ
```

今回の最小版では、次の4条件を満たす行だけを書き出す。

```text
status       = approved
consent      = confirmed
usage_scope  = internal
validity     = current
```

未確認の伝聞や過去の方針が混ざれば、文章はそれらしくても、本人らしい判断から遠ざかる。この4条件が、小さな関門になる。

## RAGはどう「本人っぽさ」を組み立てるのか

検索と回答生成を分けて見ると分かりやすい。

### 1. 検索は、似た場面を探す

質問が「安全確認の必要な新規企画を、いつ見せるか」なら、単に「企画」という言葉がある記録ではなく、次の要素が近いエピソードを探したい。

- 企画レビューという場
- 完成前の相談
- 若手から上司への共有
- 安全や法令という制約

そのため、`question` と `actual_answer` だけでなく、`decision_setting`、`situation`、`reasoning`、`exceptions`、`context_tags` も検索対象へ入れる。

### 2. LLMは、検索された判断を今の質問へ当てはめる

検索で見つかったエピソードには、過去の回答だけでなく、「なぜそうしたか」「どんな場合は例外か」がある。

LLMはそれを材料に、今の場面ならどう考えるかを文章にする。

```text
過去の判断:
  完成前に見せる。方向違いを早く見つけたいから。

過去の例外:
  安全が絡むなら、確認者と公開範囲を先に決める。

今の質問:
  安全確認が必要な新規企画を、完成前に見せてよいか。

回答:
  先に確認者と共有範囲を決め、その範囲内で3割の段階から相談する。
```

これが、「過去に同じ質問へ答えた文章を探す」のとは違うところだ。**似た状況から、優先順位と例外を取り出し、新しい状況へ適用する。**

### 3. それでも、AIが本人らしさを保証するわけではない

検索された記録が違えば、もっともらしい別の答えになる。エピソードが一件しかなければ、一度の例外を恒常的な価値観として扱うかもしれない。

だから、回答と一緒に参照元を見せる。

```text
回答
  └─ 根拠: EP-001、EP-007
        ├─ どんな場面だったか
        ├─ 実際に何と言ったか
        └─ 今も有効か
```

「本人っぽい」と感じた理由を、過去の記録まで戻って確かめられる状態が必要だ。

## Step 1：フォルダを作る

第1回のExcelを「CSV UTF-8」形式で `episodes.csv` として保存する。

```text
rag-data/
├── episodes.csv
├── csv_to_markdown.py
└── data/
    └── episodes.md  ← Pythonで生成する
```

:::message
CSVは、Excelの表を他のプログラムでも読めるようにしたテキスト形式だ。編集はExcelのまま続けてよい。
:::

## Step 2：書き出す条件をPythonへ固定する

まず重要なのは、次の関数である。

```python
def is_publishable(row):
    # 4条件をすべて満たす行だけTrueになる
    return (
        row["status"] == "approved"
        and row["consent"] == "confirmed"
        and row["usage_scope"] == "internal"
        and row["validity"] == "current"
    )
```

Excelのフィルターでも同じことはできる。ただし毎回手で操作すると、条件を一つ忘れるかもしれない。Pythonへ条件を書いておけば、誰が実行しても同じ行が選ばれる。

## Step 3：CSVをMarkdownへ変換する

`csv_to_markdown.py` を作り、次のコードを貼り付ける。

:::details csv_to_markdown.py（クリックで開く）
```python
#!/usr/bin/env python3
import argparse
import csv
from pathlib import Path

# CSVに必要な列。列名の間違いや不足もここで検出する。
REQUIRED_COLUMNS = {
    "episode_id", "status", "validity", "usage_scope", "theme",
    "decision_setting", "situation", "question", "actual_answer",
    "reasoning", "avoided_option", "outcome", "exceptions",
    "value_tags", "context_tags", "source_type", "event_date",
    "last_reviewed", "next_review_date", "consent",
    "supersedes_episode_id",
}


def clean(value):
    # Excel由来の余分な改行や空白を一つへまとめる。
    return " ".join((value or "").replace("\r", "\n").split())


def load_rows(path):
    # utf-8-sigにすると、Excelが付けるBOMの有無を吸収できる。
    with path.open("r", encoding="utf-8-sig", newline="") as handle:
        reader = csv.DictReader(handle)
        missing = sorted(REQUIRED_COLUMNS - set(reader.fieldnames or []))
        if missing:
            raise SystemExit(f"必須列がありません: {', '.join(missing)}")
        return [
            {key: clean(value) for key, value in row.items()}
            for row in reader
        ]


def is_publishable(row):
    # 未確認・利用不可・旧方針の行をRAGへ混ぜないための関門。
    return (
        row["status"] == "approved"
        and row["consent"] == "confirmed"
        and row["usage_scope"] == "internal"
        and row["validity"] == "current"
    )


def render_episode(row):
    # 質問だけでなく、場面・理由・例外も検索できる文章へ整える。
    fields = (
        ("判断した場", row["decision_setting"]),
        ("状況", row["situation"]),
        ("相談・問い", row["question"]),
        ("実際の回答・行動", row["actual_answer"]),
        ("判断理由", row["reasoning"]),
        ("避けた選択", row["avoided_option"]),
        ("結果", row["outcome"]),
        ("例外・条件", row["exceptions"]),
    )

    lines = [
        f'## {row["episode_id"]}: {row["theme"]}', "",
        f'- 最終確認日: {row["last_reviewed"]}',
        f'- 次回確認日: {row["next_review_date"]}',
        f'- 情報源種別: {row["source_type"]}',
        f'- 価値観タグ: {row["value_tags"] or "未設定"}',
        f'- 場面タグ: {row["context_tags"]}',
        f'- 置き換え元: {row["supersedes_episode_id"] or "なし"}', "",
    ]

    # source_personなど、検索に不要な管理情報は出力しない。
    for label, value in fields:
        lines.extend([f"### {label}", "", value or "記録なし", ""])

    return "\n".join(lines).rstrip()


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("input_csv", type=Path)
    parser.add_argument("output_md", type=Path)
    args = parser.parse_args()

    rows = load_rows(args.input_csv)
    sections = []
    seen_ids = set()

    for line_number, row in enumerate(rows, start=2):
        episode_id = row["episode_id"]

        # 同じIDを誤って2回登録した場合は、後の行を使わない。
        if episode_id in seen_ids:
            print(f"警告: {line_number}行目のIDが重複しています: {episode_id}")
            continue
        seen_ids.add(episode_id)

        if not is_publishable(row):
            continue

        sections.append(render_episode(row))

    header = (
        "# 承認済み・現行の判断エピソード\n\n"
        "> 収集台帳から自動生成したRAG投入用データです。"
        "修正は元のCSVで行ってください。\n\n"
    )
    output = header + "\n\n---\n\n".join(sections) + "\n"

    args.output_md.parent.mkdir(parents=True, exist_ok=True)
    args.output_md.write_text(output, encoding="utf-8")
    print(f"出力: {args.output_md} ({len(sections)}件)")


if __name__ == "__main__":
    main()
```
:::

実行する。

```bash
python3 csv_to_markdown.py episodes.csv data/episodes.md
```

ターミナルへ次のように出れば成功だ。

```text
出力: data/episodes.md (2件)
```

4件のうち、確認待ちや置き換え済みの2件は収集台帳に残ったまま、RAG投入データからは除外される。

## Step 4：何がベクトル化されるかを見る

生成されたMarkdownは、次のようになる。

```markdown
## EP-001: 新規提案

- 最終確認日: 2026-08-05
- 情報源種別: observer_interview|official_record
- 価値観タグ: 現場起点|早期検証
- 場面タグ: 完成前|方向性確認|若手から上司

### 判断した場

企画レビュー会議

### 状況

若手社員が、準備中の新規企画を完成前に見せるべきか迷っていた。

### 実際の回答・行動

完成度が3割でも見せてよい。利用者の困りごとと
検証したい仮説を1枚にして持ってきてほしい。
```

質問と回答だけではなく、テーマ、判断した場、状況、理由、例外、場面タグも一つの文章としてベクトル化する。質問の言葉が完全に一致しなくても、似た場面を探しやすくするためだ。

承認状態、同意、利用範囲、現在の有効性は、検索対象を作る前の絞り込みに使う。本格的なベクトルDBでは、これらをメタデータとして保存し、検索時の条件にもできる。

## Step 5：前回のローカルRAGへ入れる

[前回の記事](https://zenn.dev/hobomokha/articles/digital-matsumoto-local-rag)で作ったフォルダの `data/` へ、生成した `episodes.md` を置く。

```text
local-rag/
├── rag.py
└── data/
    ├── profile.md
    └── episodes.md
```

同じ質問へのRAGなし・ありの回答を比べる。

```bash
python3 rag.py --compare "新しい企画は完成してから相談した方がよいですか？"
```

次に、質問へ場面を加える。

```bash
python3 rag.py --compare "企画レビュー会議で、安全確認が必要な新規企画を相談します。完成してから見せるべきですか？"
```

検索結果だけを見る場合は、次を実行する。

```bash
python3 rag.py --show-context "企画レビュー会議で、安全確認が必要な新規企画を相談します。完成してから見せるべきですか？"
```

## 回答の差を読む

実際の文面はLLMによって変わるが、差は次のように現れる。

### RAGなし

```text
状況によります。早めに相談すると方向性を確認できる一方、
ある程度整理してから相談すると効率的です。
```

一般論としては自然だが、この人が何を重視するかは分からない。

### 薄いメモだけを入れたRAG

```text
早めに相談することが大切です。完成前でも共有するとよいでしょう。
```

少し近づいたが、「どの状態で」「何を準備して」がない。

### 判断エピソードを入れたRAG

```text
完成を待たず、3割の段階で相談します。その際は、利用者の
困りごとと検証したい仮説を1枚に整理します。ただし、法令や
安全に関わる内容は公開範囲と確認者を先に決めます。
```

回答を変えたのは、LLMの交換ではない。RAGへ渡した判断の具体性だ。

本人を知る人が最後の回答を見て「3割で見せるところも、安全なら先に確認者を置くところも、この人らしい」と感じる。その感覚が、抽象的な理念ではなく、判断の形で考え方を引き継げているかを見る手がかりになる。

確認する場所は6つある。

- 期待したエピソードが検索されたか
- 質問だけでなく、会議・完成前・安全確認という状況が効いたか
- 「3割」「仮説を1枚」のような具体的行動が入ったか
- 安全に関する例外が落ちていないか
- データにない方針を付け足していないか
- 利用者が参照元へ戻れるか

## 今回の完成条件

- 収集台帳とRAG投入データを分けた
- 4条件を満たす行だけMarkdownへ変換した
- 情報提供者名など、検索に不要な管理情報を除外した
- 質問だけの場合と、場面を加えた場合を比較した
- RAGなし／薄いメモ／判断エピソードの差を確認した
- どの検索結果が「本人っぽい」回答につながったか説明できた

これで、場面データが検索を変え、検索が回答を変えるところまで体験できた。

次の記事では、判断が変わったときに古いデータをどう置き換えるか、そして「正しいか」と「本人っぽいか」をどう分けて評価するかを扱う。

[次の記事：作って終わりにしない――意思決定RAGを更新・評価する](https://zenn.dev/hobomokha/articles/decision-rag-update-and-evaluate)

## 参考資料

- [DigitalMATSUMOTO 公開リポジトリ](https://github.com/m07takash/DigitalMATSUMOTO)
- [Digital MATSUMOTO（公式サイト）](https://www.digitalmatsumoto.com/ja/digital-matsumoto/)
