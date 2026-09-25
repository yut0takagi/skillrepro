# skillrepro 設計仕様

エージェントのスキルが「何回実行しても、どのモデルでも、同じように振る舞うか」を検査するツール。
スキルは YAML で手順を構造化して書き、その構造を使って実行のブレを手順単位で特定する。

## 1. 解く問題

スキルは量産されるが、共有すると動かなくなる。原因は2種類ある。

- **振る舞いのブレ**: 同じ依頼でも、実行ごと・モデルごとに手順を飛ばしたり、別の道筋を通ったりする
- **環境の違い**: 作者の手元にしかないパス・CLI・MCP・環境変数に黙って依存している

既存ツールはどちらにも十分に答えていない。

| 既存 | できること | 足りないこと |
| --- | --- | --- |
| skill-check / skill-linter / skills-lint / skill-lint | SKILL.md の静的検査（形式・品質・読み込み失敗・セキュリティ） | 実行しない。振る舞いのブレは測れない |
| Anthropic `skill-creator` | スキルあり・なしを複数回実行し、合格率を平均 ± 標準偏差で集計 | 最終出力しか見ないので、どの手順でブレたか分からない。会話の中で回すので CI に載らない。副作用を隔離しない。モデルを並べて比べられない |

skillrepro の中心は **振る舞いの再現性を、手順単位で測ること** に置く。
手順を YAML で構造化させるのは、そのための手段である。静的検査と環境検査は、測定の前提条件として持つ。

## 2. 全体像

```text
skill.yaml ──compile──▶ SKILL.md（本番用 / 観測用）
     │
     └─ evals ──▶ run: eval × モデル × N回（Claude Agent SDK でヘッドレス実行）
                    │  副作用のある能力はスタブに差し替え、作業は一時ディレクトリで行う
                    ▼
                 trace: 手順ごとの「入ったか / 呼んだツール / gate / 出力」
                    ▼
                 report: ① 合格率とブレ（eval × モデル）
                         ② 手順ごとの一致率と、最初に分岐した手順
                         ③ 前回の実行（または旧版）との差分
```

## 3. スキル形式（skill.yaml）

Agent Skills の上位互換にする。トップレベルの `name` / `description` / `allowed-tools` などは
SKILL.md の frontmatter と同じキーで、そのまま出力される。
構造は「レビューと観測の単位」にだけ置き、各項目の中身は自由な文章で書く。

```yaml
spec: skillrepro/v0.1

# Agent Skills 互換（SKILL.md の frontmatter になる）
name: weekly-minutes-digest
description: |
  直近1週間の議事録を読み、決定事項とネクストアクションをまとめて Slack に投稿する。
  Use when: 「今週の議事録まとめて」「週次ダイジェスト」
allowed-tools: [Read, Glob, Bash]   # requires.mcp の能力は compile が解決後のツール名で追記する

# 拡張（SKILL.md では本文に展開される）
purpose: |
  会議が多い週に決定事項が埋もれるのを防ぐ。

inputs:
  - id: period
    about: 対象期間。省略時は直近7日。

requires:                       # 環境への依存はすべてここに宣言する
  tools:
    - python3: "同梱スクリプトの実行"
  mcp:
    - slack.post:
        about: "まとめを投稿する"
        default: mcp__slack__slack_post_message   # 任意。local が無いときに使う
  env:
    - SLACK_CHANNEL: "投稿先チャンネル ID"
  paths:
    - minutes_dir: "議事録が置かれているフォルダ"
  files:
    - path: scripts/list_minutes.py
      use: run                  # run = 実行する / read = 参照として読む

permissions:                    # 副作用の宣言。スタブ化と lint の根拠になる
  reads:  [paths.minutes_dir]
  writes: []
  sends:  [slack.post]

steps:
  - id: collect
    do: |
      {{paths.minutes_dir}} から期間内の議事録を集める。
    run: python3 scripts/list_minutes.py --dir {{paths.minutes_dir}} --days 7
    freedom: low                # low のときは run が必須。compile が「コマンドを変えない」旨を添える
    why: 要約に失敗した空の議事録を、スクリプト側で確実に除外するため。
  - id: summarize
    do: |
      決定事項とネクストアクションを抜き出す。担当者が書かれていないものは「担当未定」とする。
    freedom: high
    verify: |
      すべての決定事項に、出典の議事録ファイル名が付いている。
    on_fail: summarize
  - id: post
    do: まとめを {{env.SLACK_CHANNEL}} に投稿する。
    uses: [slack.post]
    gate: confirm               # 実行前に人の承認を取る

done_when: |
  投稿の URL を返し、対象にした議事録の一覧を添える。

evals:
  - id: normal-week
    query: 今週の議事録まとめて
    files: [fixtures/minutes/]
    stubs:
      slack.post: { returns: { ok: true, url: "https://example.slack.com/p1" } }
    gate_answer: approve
    assertions:                 # 機械的に判定するもの（優先）
      - step_order: [collect, summarize, post]
      - tool_called: { capability: slack.post, times: 1 }
      - gate_reached: post
    expected_behavior:          # LLM judge が判定するもの
      - 空の議事録（fixtures/minutes/empty.md）がまとめに含まれていない
```

### 3.1 環境ごとの値（skill.local.yaml）

利用者それぞれが持ち、共有しない（gitignore 対象）。
`compile` と `run` は skill.yaml とこれを合成する。

```yaml
paths:
  minutes_dir: 仕事/議事録
mcp:
  slack.post: mcp__slack__slack_post_message
env:
  SLACK_CHANNEL: C0XXXXXXXXX
```

- `mcp` の能力名は作者が自由に付ける。v0.1 では語彙を標準化しない
- local に割り当てが無く `default` がある能力は `default` を使う。そのとき `doctor` と `run` は必ず「既定値を使用中」と表示する
- local にも `default` にも無い能力がある場合、`compile` は SKILL.md を出力せずに終了する

## 4. 振る舞いの再現性の測定（中心機能）

### 4.1 実行（run）

- 実行には Claude Agent SDK（TypeScript）を使い、ヘッドレスで動かす
- 組み合わせは `eval × モデル × N回`。既定は N=5、モデルは設定ファイルで指定する（既定は Haiku・Sonnet・Opus の3つ）
- 1回の実行ごとに一時ディレクトリを作り、`files` をコピーしてから始める。実行後に消す（`--keep` で残せる）
- 同時実行数は `--concurrency` で指定する（既定 4）

### 4.2 副作用の隔離

同じスキルを何十回も走らせるので、本物の副作用は起こさない。

| 対象 | 隔離の方法 |
| --- | --- |
| `permissions.sends` / `writes` にある MCP 能力 | 同じ名前のスタブツールを持つ MCP サーバーをプロセス内で立て、本物のツールは `disallowedTools` で塞ぐ。スタブは呼び出しを記録し、eval の `stubs` に書いた値を返す |
| `permissions.sends` / `writes` に書いた CLI（例: `writes: [tools.bq]`） | PATH の先頭に shim を置き、呼び出しを記録して `stubs` の値を返す |
| ファイル書き込み | 一時ディレクトリの中に限る |
| スタブが定義されていない副作用 | 実行を止め、その回を `error: unstubbed-side-effect` とする。本物には流さない |

### 4.3 手順の観測（trace）

観測用に compile した SKILL.md は、各手順の開始時に目印 `[step:<id>]` を、
gate に達したときに `[gate:<id>]` を出力するよう指示する。
SDK が記録するツール呼び出しを、目印の区間で切り分けて手順に割り当てる。

1回分の trace には次を記録する。

- 通過した手順の列（例: `collect → summarize → summarize → post`）
- 手順ごとに呼んだツールと引数、実行したコマンド
- gate に達したか、それに runner がどう答えたか
- 最終出力、トークン数、所要時間

目印が出なかった手順は「飛ばした」とみなす。目印を飛ばすこと自体が、再現性の低さのシグナルになる。

gate の扱い: 観測用 SKILL.md は gate で止まって承認を求めるよう指示される。
runner は `[gate:<id>]` を受けたら、eval の `gate_answer`（`approve` / `deny`、既定は `deny`）で応答する。

### 4.4 判定

- **assertions（決定的）**: `step_order` / `step_visited` / `tool_called` / `gate_reached` / `file_exists` / `output_matches`（正規表現）
- **expected_behavior（LLM judge）**: 各項目を合格・不合格で判定させ、根拠を記録する。judge のモデルは設定で固定する
- 両者は別の列で集計する。judge 自身のブレがスキルのブレと混ざらないようにするため

### 4.5 指標

| 指標 | 定義 |
| --- | --- |
| 合格率 | eval × モデルごとの、N回中の合格回数 / N |
| flaky | 合格率が 0 より大きく 1 より小さい eval × モデル |
| 経路一致率 | 最頻の手順列と完全に一致した実行の割合 |
| 最初の分岐点 | 最頻の手順列から最初に外れた手順 id と、その回数 |
| 手順ごとのツール一致率 | 手順ごとに、呼んだツール集合が最頻の集合と一致した実行の割合 |
| 前回との差 | 保存済みの前回結果（または `--baseline <旧版の skill.yaml>`）との、上記すべての差 |

### 4.6 レポート

ターミナル表示の例:

```text
weekly-minutes-digest  eval: normal-week  N=5

              合格率   経路一致  最初の分岐
  haiku        2/5      40%     summarize（3回: verify に失敗したまま post に進まず終了）
  sonnet       5/5     100%     -
  opus         5/5     100%     -

  手順ごとのツール一致率
    collect    100% / 100% / 100%
    summarize   40% / 100% / 100%
    post       100% / 100% / 100%
```

- `--format json` で機械可読な結果を出す。GitHub Action はこれを PR の注釈に変換する
- 結果は `.skillrepro/runs/<実行ID>/` に保存し、次回の「前回との差」に使う

### 4.7 コスト上限

- `run` は実行前に「eval 数 × モデル数 × N回 = 合計回数」と推定コストを表示する
- 推定がしきい値（設定 `budget.usd`、既定 $5）を超える場合は、`--yes` が無ければ実行しない
- 実行中に実測コストがしきい値を超えたら、残りの実行を止めてそこまでの結果を出す

## 5. 前提条件の検査

振る舞いを測る前に、環境が揃っていることを確かめる。
揃っていない環境で測ると、ブレの原因が環境なのかスキルなのか切り分けられないため。
`run` は開始前に `lint` と `doctor` を内部で実行し、エラーがあれば測定に進まない。

### 5.1 lint（静的検査）

1ルール = 1ファイルで実装する。各ルールは「どこで・何が・どう直すか」を出す。

Agent Skills とベストプラクティスの検査（生成後の SKILL.md に対して行う）:

- `name`: 64文字以内、小文字・数字・ハイフンのみ、XML タグなし、`anthropic` / `claude` を含まない
- `description`: 空でない、1,024文字以内、XML タグなし、「何をするか」と「いつ使うか」の両方を含む、一人称・二人称で書かない
- 本文が500行未満
- 参照ファイルは SKILL.md から1階層まで。100行を超える参照ファイルには先頭に目次がある
- バックスラッシュのパスがない
- MCP ツールが出力先の形式の完全修飾名になっている

YAML 層の検査（skill.yaml に対して行う）:

- 未宣言の依存: 本文・`run` に、`requires` に無い CLI・MCP 名・環境変数・パスが出てこない。`/Users/` や `~/` で始まる固定パスがない
- 同梱漏れ: 参照しているファイルが `requires.files` にあり、実在する
- `sends` / `writes` / 削除系の操作を含む手順に `gate` がある
- `freedom: low` の手順に `run` がある
- `on_fail` の参照先が存在する手順 id である
- `evals` が3件以上ある（3件未満は警告）
- `sends` / `writes` の能力すべてに、各 eval の `stubs` がある（無ければ警告。`run` 時は 4.2 の規則で止まる）

意味の判断が要るもの（簡潔さ、用語の一貫性、時間依存の記述、選択肢の多さ）は lint で合否を出さない。
`review` コマンドが観点のチェックリストを Markdown で出力するだけにする。

### 5.2 doctor（環境検査）

`requires` がこの環境で満たせるかを確かめる。

- CLI が PATH にあるか
- MCP の割り当て先ツールが接続済みか（既定値を使っている場合はそう表示する）
- 環境変数が設定されているか。値は表示せず、設定の有無だけ出す
- パスが存在するか
- 足りない項目には「何をすれば直るか」を添える

## 6. compile（SKILL.md の生成）

- `--target claude-code | claude-api | codex` で出力先を選ぶ。能力名はそれぞれの形式のツール名に変換する（Claude Code は `mcp__server__tool`、Claude API は `Server:tool`）
- `--trace` を付けると観測用（4.3 の目印指示入り）を出す。付けないと本番用
- 出力の先頭に「skill.yaml から生成。手編集禁止」のコメントを入れる
- `steps` は番号付きの手順と、進捗チェックリストとして展開する（公式のワークフローパターンに合わせる）

## 7. CLI

| コマンド | 役割 |
| --- | --- |
| `init` | skill.yaml と skill.local.yaml の雛形を作る。`.gitignore` への追記は確認してから行う |
| `lint [path]` | 5.1 の検査 |
| `doctor [path]` | 5.2 の検査 |
| `compile [path] --target <t> [--trace]` | 6 の生成 |
| `run [path] [--models ...] [-n N] [--baseline <path>]` | 4 の測定 |
| `report [実行ID]` | 保存済みの結果を表示する |
| `import <SKILL.md>` | 雛形 YAML を作る。推定できない欄（`requires` / `permissions` / `why` / `evals`）は `未記入` のまま残し、lint が警告する |
| `review [path]` | 5.1 末尾の観点チェックリストを出力する（合否なし） |

終了コード: エラーがあれば 1。`run` は flaky な eval × モデルが1つでもあれば 1（`--allow-flaky` で 0）。

## 8. リポジトリ構成

```text
skillrepro/
├── spec/
│   ├── SPEC.md               # 形式の仕様（人が読む正本）
│   └── schema.v0.1.json      # JSON Schema
├── src/
│   ├── parse/                # YAML → 型付きモデル（行番号を保持）
│   ├── rules/                # lint ルール（1ルール = 1ファイル）
│   ├── targets/              # compile の出力先 adapter
│   ├── doctor/
│   ├── runner/               # Agent SDK 実行・スタブ・一時ディレクトリ
│   ├── trace/                # 目印とツールログの対応づけ
│   ├── metrics/              # 4.5 の指標
│   ├── report/
│   ├── import/
│   └── cli.ts
├── examples/                 # lint を通り、run できる見本スキル3本
└── action.yml                # GitHub Action
```

## 9. 実装の順序

中心機能に早く到達する順にする。

1. スキーマと parse、`compile`（本番用・観測用）
2. `run` / trace / 指標 / レポート（中心機能）。この時点では lint と doctor は最小限（スキーマ検査のみ）
3. lint の全ルール
4. doctor
5. import / review / GitHub Action

## 10. テスト方針

- lint: ルールごとに「通る YAML」と「落ちる YAML」の fixture を置き、期待するエラーをスナップショットで比較する
- compile: examples の出力をスナップショットで固定する
- trace と指標: 記録済みのツールログ（fixture）から手順列と指標を計算し、期待値と比較する。LLM は呼ばない
- runner: スタブと一時ディレクトリの隔離を、本物の SDK を呼ばない偽のエージェントで検証する。スタブ未定義の副作用で必ず止まることを確認する
- 移植性の回帰: 固定パス・MCP 名の直書き・同梱漏れなど、実在するスキルで見つかった依存パターンの最小例を fixture にし、すべて検出できることを確認する
- 実モデルを使う end-to-end は examples に対してだけ、手動または定期実行で回す（CI の毎回の実行には入れない）

## 11. v0.1 でやらないこと

- Claude 以外のエージェント（Codex CLI など）での実行。compile の出力先としては持つが、`run` は Claude Agent SDK のみ
- 能力名の標準語彙
- スキルの自動修正
