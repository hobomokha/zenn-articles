---
title: "Excelの判断記録をAIに読ませる――RAG投入データをPythonで作る"
emoji: "🔧"
type: "tech"
topics: ["rag", "python", "excel", "llm"]
published: false
---

> 連載「小さく作る意思決定RAG」の第2回です。
> [第1回](https://zenn.dev/hobomokha/articles/decision-rag-collect-episodes)では、本人・周囲・公式記録から判断エピソードを4件集め、Excelへ整理しました。今回は、そのExcelからRAGへ入れてよい行だけを取り出します。

## 今回のゴール

この記事でやることは一つだけです。

**Excelの収集台帳から、承認済みかつ現在有効な判断だけをMarkdownへ変換し、前回作ったローカルRAGで回答の差を比べる。**

完成すると、次の流れを手元で確認できます。

```text
Excel / CSV
  │ 承認・同意・利用範囲・有効性を確認
  ▼
RAG投入用Markdown
  │ 場面・質問・回答・理由・例外をベクトル化
  ▼
質問 + 現在の状況から検索
  ▼
根拠を添えて回答
```

## なぜExcelをそのままRAGへ入れないのか

収集台帳には、情報提供者の名前、確認前の伝聞、非公開の補足、古くなった判断が含まれます。全部をベクトルDBへ入れる必要はありません。

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

今回の最小版では、次の4条件を満たす行だけを書き出します。

```text
status       = approved
consent      = confirmed
usage_scope  = internal
validity     = current
```

これが、未確認情報や過去の方針をRAGへ混ぜないための小さな関門になります。

## Step 1：フォルダを作る

第1回のExcelを「CSV UTF-8」形式で `episodes.csv` として保存します。

```text
rag-data/
├── episodes.csv
├── csv_to_markdown.py
└── data/
    └── episodes.md  ← Pythonで生成する
```

:::message
CSVは、Excelの表を他のプログラムでも読めるようにしたテキスト形式です。Excelで編集を続けても構いません。
:::

## Step 2：書き出す条件をPythonへ固定する

まず重要なのは、次の関数です。

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

Excelのフィルターでも同じことはできます。ただし毎回手で操作すると、条件を一つ忘れるかもしれません。Pythonへ条件を書いておけば、誰が実行しても同じ判断になります。

## Step 3：CSVをMarkdownへ変換する

`csv_to_markdown.py` を作り、次のコードを貼り付けます。

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

実行します。

```bash
python3 csv_to_markdown.py episodes.csv data/episodes.md
```

ターミナルへ次のように出れば成功です。

```text
出力: data/episodes.md (2件)
```

4件のうち、確認待ちや置き換え済みの2件は収集台帳に残ったまま、RAG投入データからは除外されています。

## Step 4：何がベクトル化されるかを見る

生成されたMarkdownは、次のようになります。

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

質問と回答だけではなく、テーマ、判断した場、状況、理由、例外、場面タグも一つの文章としてベクトル化します。質問の言葉が完全に一致しなくても、似た場面を探しやすくするためです。

一方、承認状態、同意、利用範囲、現在の有効性は、検索対象を作る前の絞り込みに使っています。本格的なベクトルDBでは、これらをメタデータとして保存し、検索時の条件にもできます。

## Step 5：前回のローカルRAGへ入れる

[前回の記事](https://zenn.dev/hobomokha/articles/digital-matsumoto-local-rag)で作ったフォルダの `data/` へ、生成した `episodes.md` を置きます。

```text
local-rag/
├── rag.py
└── data/
    ├── profile.md
    └── episodes.md
```

同じ質問へのRAGなし・ありの回答を比べます。

```bash
python3 rag.py --compare "新しい企画は完成してから相談した方がよいですか？"
```

次に、質問へ場面を追加します。

```bash
python3 rag.py --compare "企画レビュー会議で、安全確認が必要な新規企画を相談します。完成してから見せるべきですか？"
```

検索結果だけを見る場合は、次を実行します。

```bash
python3 rag.py --show-context "企画レビュー会議で、安全確認が必要な新規企画を相談します。完成してから見せるべきですか？"
```

## 回答の差を読む

実際の文面はLLMによって変わりますが、差は次のように現れます。

### RAGなし

```text
状況によります。早めに相談すると方向性を確認できる一方、
ある程度整理してから相談すると効率的です。
```

一般論としては自然ですが、この組織でどう動くべきかは分かりません。

### 薄いメモだけを入れたRAG

```text
早めに相談することが大切です。完成前でも共有するとよいでしょう。
```

少し近づきましたが、「どの状態で」「何を準備して」がありません。

### 判断エピソードを入れたRAG

```text
完成を待たず、3割の段階で相談します。その際は、利用者の
困りごとと検証したい仮説を1枚に整理します。ただし、法令や
安全に関わる内容は公開範囲と確認者を先に決めます。
```

回答を変えたのは、LLMの交換ではありません。RAGへ渡したデータの具体性です。

確認するポイントは次の6つです。

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

これで、データを変えると検索結果と回答が変わることを体感できました。

次の記事では、判断が変わったときに古いデータをどう置き換えるか、そして更新によってRAGが壊れていないかをどう評価するかを扱います。

[次の記事：作って終わりにしない――意思決定RAGを更新・評価する](https://zenn.dev/hobomokha/articles/decision-rag-update-and-evaluate)

## 参考資料

- [DigitalMATSUMOTO 公開リポジトリ](https://github.com/m07takash/DigitalMATSUMOTO)
- [Digital MATSUMOTO（公式サイト）](https://www.digitalmatsumoto.com/ja/digital-matsumoto/)
