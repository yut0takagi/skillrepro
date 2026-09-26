# skillrepro 設計仕様

エージェントのスキルが「何回実行しても、どのモデルでも、同じように振る舞うか」を検査するツール。
スキルは YAML で手順を構造化して書き、その構造を使って実行のブレを手順単位で特定する。

## 1. 解く問題

スキルは量産されるが、共有すると動かなくなる。原因は2種類ある。

- **振る舞いのブレ**: 同じ依頼でも、実行ごと・モデルごとに手順を飛ばしたり、別の道筋を通ったりする
- **環境の違い**: 作者の手元にしかないパス・CLI・MCP・環境変数・agent 定義に黙って依存している

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
                    │  外部とのやり取りは既定ですべてスタブ。作業は一時ディレクトリで行う
                    ▼
                 trace: 通った route・手順・周回・分岐先・呼んだツール・出力
                    ▼
                 report: ① 合格率とブレ（eval × モデル）
                         ② 構造ごとの一致率と、最初に分岐した箇所
                         ③ 前回の実行（または旧版）との差分
```

## 3. スキル形式（skill.yaml）

### 3.1 原則

- **Agent Skills の上位互換**: トップレベルの `name` / `description` / `allowed-tools` などは SKILL.md の frontmatter と同じキーで、そのまま出力される
- **すべての節を `name` と `description` で書く**: スキル・route・step のどれも、`name`（識別子）と `description`（自然言語の説明）を持つ。その上に、種類ごとのキーを足す
- **条件は自然言語のまま書き、判断はエージェントに任せる。ただし、選んだ結果は必ず目印で報告させる**: ツールは自然言語の条件を評価しない。エージェントの選択を目印で観測し、eval に書いた期待と照合する（4.3）
- 構造は「レビューと観測の単位」にだけ置く。各節の中身は自由な文章で書く

### 3.2 名前の規則

- `name` はスキル・route・step のすべてで同じ規則にする: 64文字以内、小文字・数字・ハイフンのみ
- スキルには任意で `title`（表示名。日本語可）を付けられる
- 呼び出し名を `name` と別にしたい場合（例: Claude Code で日本語名のまま `/日報` と呼びたい）は `invoke_as` を書く。`compile --target claude-code` はこれをディレクトリ名に使う
- `name` はスキルの中で一意であること（route と step をまたいで重複しない）

### 3.3 トップレベル

```yaml
spec: skillrepro/v0.1

# Agent Skills 互換（SKILL.md の frontmatter になる）
name: managing-tasks
title: タスク追加
invoke_as: タスク追加
description: |
  タスクの追加・完了・一覧を、ガントチャート・ダッシュボード・詳細 md の3層で管理する。
  Use when: 「タスクに入れて」「終わった」「何が残ってる」
allowed-tools: [Read, Edit, Write, Glob]   # requires.mcp の能力は compile が解決後のツール名で追記する

# 拡張
purpose: |                # なぜこのスキルがあるか
inputs:                   # 受け取るもの（name / description）
requires:                 # 環境への依存（3.6）
permissions:              # 外部とのやり取りの宣言（3.7）
knowledge:                # 手順に属さない参照知識（3.8）
rules:                    # 全手順に効く不変条件（3.8）
routes:                   # モード分岐（3.5）。routes か steps のどちらか一方を持つ
steps:                    # 手順（3.4）
done_when: |              # 完了条件
evals:                    # 評価（4.4）
```

### 3.4 手順（step）

すべての step は `name` と `description` を持ち、次の種類キーを**高々1つ**持つ。
種類キーが無い step は、`description` に書いた作業をエージェントが行う。

| 種類キー | 意味 |
| --- | --- |
| （なし） | `description` に書いた作業を行う |
| `run` | 正確なシェルコマンドを実行する |
| `call` | 引数を固定して MCP を呼ぶ。`{capability, args}` |
| `ask` | ユーザーに質問する。`{question, choices?}`。`choices` が無ければ自由記述 |
| `dispatch` | subagent に委譲する。`{agent, inputs, outputs}` |
| `loop` | 繰り返す。`{max, until, steps, on_exhausted}`。`on_exhausted` は抜けられなかったときに行う step（name / description を持つ） |
| `foreach` | 集合の要素ごとに実行する。`{over, as, steps}` |
| `parallel` | 同時に実行すべき step の束。値は step のリスト |
| `decide` | 多分岐の行き先を決める。`{cases: [{description, goto}]}` |

すべての step に付けられる共通キー:

| キー | 意味 |
| --- | --- |
| `when` | 自然言語の条件。満たさないときは飛ばす |
| `when_expr` | 機械で評価する条件。前の step の出力を使う（例: `steps.dryrun.outputs.gb > 100`）。runner が評価する |
| `optional` | 飛ばしても「手順抜け」と数えない |
| `freedom` | `high` / `medium` / `low`。`low` のときは `run` か `call` が必須 |
| `why` | この手順がある理由 |
| `reads` | この手順で読む `knowledge` や同梱ファイル |
| `rules` | この手順で特に守る `rules` の `name` |
| `outputs` | 後の step から `{{steps.<name>.outputs.<key>}}` で参照する値（`{key: description}`） |
| `verify` | 結果の確かめ方（自然言語）。`on_fail` と組で使う |
| `on_fail` | `verify` に失敗したときの行き先の step `name` |
| `on_error` | 実行に失敗したときの扱い。`{then: stop / continue / goto, goto?, say?}` |
| `uses` | この手順で使う能力（`permissions` の能力名） |
| `gate` | 実行前の承認。3.7 を参照 |

`goto` と `on_fail` の行き先は、同じ route の中、または同じ `loop` の中の step に限る。

### 3.5 モード分岐（routes）

```yaml
routes:
  - name: add
    description: |
      タスクを新しく追加する。
      Use when: 新しい作業を「タスクに入れて」「後でやる」と言っている。
    steps:
      - name: match-project
        description: 依頼内容から PJT を判定する。
        reads: [knowledge.projects]
      - name: confirm-project
        description: PJT が判定できないときだけ確認する。
        when: どの PJT にも合致しない
        ask:
          question: どの PJT のタスクですか？
          choices: "{{local.data.projects}}"
      - name: write-task
        description: ガントと詳細 md に追記する。既存のタスクは上書きしない。
        rules: [no-overwrite]
  - name: complete
    description: |
      タスクを完了にする。
      Use when: 「終わった」「完了」と言っている。
    steps: [...]
```

route の選び方は、スキルの description と同じ書き方（何をするか＋Use when）で `description` に書く。

### 3.6 依存の宣言（requires）

```yaml
requires:
  tools:                       # CLI
    - python3: "同梱スクリプトの実行"
  mcp:                         # 能力名で宣言する
    - slack.post:
        description: "まとめを投稿する"
        default: mcp__slack__slack_post_message   # 任意。local が無いときに使う
  env:
    - SLACK_CHANNEL: "投稿先チャンネル ID"
  paths:
    - minutes_dir: "議事録が置かれているフォルダ"
  values:                      # パスでも env でもない環境固有の値（プロジェクト ID など）
    - bq_project: "BigQuery のプロジェクト ID"
  files:                       # 同梱ファイル
    - path: scripts/list_minutes.py
      use: run                 # run = 実行する / read = 参照として読む
  agents:                      # 委譲先の subagent 定義
    - slide-adversary-logic: "筋道の敵対レビュー"
  rules:                       # スキル外のルールファイル
    - bq-cost-guard: "BigQuery のコストガード"
  services:                    # 起動している必要があるサービス
    - open-design:
        description: "HTML デッキの生成"
        check: "curl -sf {{values.open_design_url}}/health"
  auth:                        # どのアカウントで認証されている必要があるか
    - gcloud:
        check: "gcloud config get-value account"
        expect: "@example\\.com$"
  resources:                   # 実在する必要がある外部リソース
    - category_master:
        description: "商品カテゴリマスタ（日付つきスナップショット）"
        check: "bq ls {{values.bq_project}}:retail"
```

本文では `{{paths.minutes_dir}}` `{{values.bq_project}}` のように名前で参照する。具体的な値はスキルに書かない。

### 3.7 外部とのやり取り（permissions）と gate

```yaml
permissions:
  reads:  [paths.minutes_dir, calendar.list]   # 読むもの
  writes: [paths.tasks_dir]                    # 書くもの
  sends:  [slack.post]                         # 外部に送るもの
  bills:  [bq.query]                           # 課金されるもの
```

- `run` の測定では、ここに書いた能力と CLI は**すべて既定でスタブ**にする（4.2）
- 能力には CLI も書ける（例: `bills: [tools.bq]`）

gate の書き方:

```yaml
gate: confirm                  # 実行前に承認を取る
gate: none                     # 承認を取らない。why が必須
gate:                          # 条件つき・多段
  - when_expr: steps.dryrun.outputs.gb > 400
    kind: refuse               # 実行しない
  - when_expr: steps.dryrun.outputs.gb > 100
    kind: confirm_strict       # 曖昧な返答を承認とみなさない
  - when_expr: steps.dryrun.outputs.gb > 5
    kind: confirm
```

条件つき gate は上から順に評価し、最初に当てはまったものを使う。どれにも当てはまらなければ承認なしで進む。

### 3.8 知識と規則

```yaml
knowledge:
  - name: projects
    description: PJT の一覧と、依頼文から判定するときの手がかり
    placement: inline          # inline = 本文に展開 / file = 同梱ファイルにして SKILL.md から参照
    content: |
      ...
rules:
  - name: no-overwrite
    description: 既存のタスク行は書き換えず、追記だけする。
  - name: duration-no-round
    description: Duration 型に直接 `.round()` を呼ばない。先に数値へ変換する。
    check: "node scripts/check_duration_round.js {{file}}"   # 任意。機械で判定できる規則だけ
```

- `rules` は特定の step に属さない不変条件。step の `rules:` と assertion（4.4）から `name` で参照できる
- `knowledge` のうち、環境ごとに違う値（PJT 一覧など）は skill.local.yaml の `data:` に置き、`{{local.data.<key>}}` で参照する

### 3.9 環境ごとの値（skill.local.yaml）

利用者それぞれが持ち、共有しない（gitignore 対象）。`compile` と `run` は skill.yaml とこれを合成する。

```yaml
paths:
  minutes_dir: 仕事/議事録
values:
  bq_project: my-gcp-project
mcp:
  slack.post: mcp__slack__slack_post_message
env:
  SLACK_CHANNEL: C0XXXXXXXXX
data:
  projects: [project-a, project-b, project-c]
```

- 能力名は作者が自由に付ける。v0.1 では語彙を標準化しない
- local に割り当てが無く `default` がある能力は `default` を使う。そのとき `doctor` と `run` は必ず「既定値を使用中」と表示する
- 必要な値が local にも `default` にも無い場合、`compile` は SKILL.md を出力せずに終了する

## 4. 振る舞いの再現性の測定（中心機能）

### 4.1 実行（run）

- 実行には Claude Agent SDK（TypeScript）を使い、ヘッドレスで動かす
- 組み合わせは `eval × モデル × N回`。既定は N=5、モデルは設定ファイルで指定する（既定は Haiku・Sonnet・Opus の3つ）
- 1回の実行ごとに一時ディレクトリを作り、eval の `files` をコピーしてから始める。実行後に消す（`--keep` で残せる）
- 同時実行数は `--concurrency` で指定する（既定 4）

### 4.2 スタブ

同じスキルを何十回も走らせるので、外部とのやり取りは既定ですべてスタブにする。本物を使うのは `--live` を明示したときだけ。

| 対象 | 方法 |
| --- | --- |
| `permissions` にある MCP 能力（reads / writes / sends / bills のすべて） | 同じ名前のスタブツールを持つ MCP サーバーをプロセス内で立て、本物のツールは `disallowedTools` で塞ぐ |
| `permissions` にある CLI | PATH の先頭に shim を置く |
| `dispatch` の subagent | `stubs.agents.<名前>` があれば、subagent を起動せずに固定の出力を返す |
| ファイルの読み書き | 一時ディレクトリの中に限る |
| スタブが定義されていないやり取り | その回を止めて `error: unstubbed-call` とする。本物には流さない |

スタブの返り値は、呼び出しの内容で出し分けられる。

```yaml
stubs:
  bq.query:
    - match: { args.dryRun: true }
      returns: { totalBytesProcessed: 2147483648 }
    - match: { args.query: "/COUNT\\(\\*\\)/" }
      returns_file: fixtures/bq/count.json
    - returns_file: fixtures/bq/rows.json      # match なし = それ以外すべて
  calendar.list:
    - returns_file: fixtures/calendar/2026-09-25.json
  agents:
    slide-adversary-logic:
      returns_file: fixtures/agents/logic-review-ng.md
```

`match` の値が `/…/` のときは正規表現として扱う。上から順に照合し、最初に当てはまった返り値を使う。

### 4.3 観測（trace）

観測用に compile した SKILL.md は、次の目印を出力するよう指示する。

| 目印 | 出すとき |
| --- | --- |
| `[route:<name>]` | route を選んだとき |
| `[step:<name>]` | step を始めたとき |
| `[skip:<name> <理由>]` | `when` を満たさず、または `optional` で飛ばしたとき |
| `[ask:<name>]` | 質問したとき |
| `[dispatch:<name> <agent>]` | subagent に委譲したとき |
| `[round:<name> <n>]` | loop の n 周目に入ったとき |
| `[item:<name> <key>]` | foreach の要素に入ったとき |
| `[decide:<name> <goto>]` | decide で行き先を選んだとき |
| `[gate:<name> <kind>]` | gate に達したとき |

SDK が記録するツール呼び出しを、目印の区間で切り分けて step に割り当てる。
目印が出なかった step は「飛ばした」とみなす（`optional` と、`[skip]` を出したものを除く）。

runner の応答:

- `ask` と `gate` に達したら、runner は eval の `replies` の値で応答する
- `replies` に無い `ask` に達したら、その回を止めて `error: unanswered-ask` とする
- `replies` に無い `gate` の既定の応答は却下

### 4.4 評価（evals）

evals もほかの節と同じく `name` と `description` を持つ。

```yaml
evals:
  - name: add-unknown-project
    description: PJT が判定できない依頼では、書き込む前に確認する
    query: 「来週までに資料の叩きを作る」をタスクに入れて
    files: [fixtures/tasks/]
    clock: 2026-09-25T09:00:00+09:00           # 日付・時刻の固定
    stubs: {}
    replies:
      confirm-project: project-a
    assertions:                                 # 機械的に判定するもの（優先）
      - route_selected: add
      - asked: confirm-project
      - step_order: [match-project, confirm-project, write-task]
      - file_matches: { path: 仕事/タスク/ガント.mw, pattern: "資料の叩き", count: 1 }
      - rule_holds: no-overwrite
    expected_behavior:                          # LLM judge が判定するもの
      - タスクの期限が 2026-10-02 になっている
```

assertion の一覧:

| assertion | 判定すること |
| --- | --- |
| `route_selected` | 選んだ route |
| `step_order` | 通った step の順序（loop・foreach・parallel の扱いは 4.5） |
| `step_visited` / `step_not_visited` | 特定の step を通った / 通らなかった |
| `asked` / `not_asked` | 特定の ask を出した / 出さなかった |
| `rounds` | loop の周回数（`{loop, min?, max?}`） |
| `dispatched` | subagent に委譲した（`{agent, times?}`） |
| `parallel_fired` | parallel の束を同時に発火した（束の中のツール呼び出しが、どれかの完了を待たずに始まった） |
| `tool_called` | 能力や組み込みツールを呼んだ（`{capability または tool, times?, args_match?}`） |
| `gate_reached` | gate に達した（`{step, kind?}`） |
| `file_exists` | ファイルがある（`{path または glob}`） |
| `file_matches` | ファイルの内容が正規表現に一致する（`{path, pattern, count?}`） |
| `file_check` | ファイルに検証コマンドを当てて成功する（`{path, run}`） |
| `output_matches` | 最終出力が正規表現に一致する |
| `rule_holds` | `rules` の規則が守られた。規則に `check` があればそれで判定し、無ければ LLM judge に回す |

- `expected_behavior` と、`check` の無い `rule_holds` は LLM judge が判定する。judge のモデルは設定で固定する
- 決定的な判定と judge の判定は、別の列で集計する。judge 自身のブレがスキルのブレと混ざらないようにするため

### 4.5 指標

構造ごとに比べ方を変える。

| 構造 | 比べ方 |
| --- | --- |
| routes | まず「どの route を選んだか」の一致率を出す。経路の比較は route ごとに行う |
| loop | 周回数は別の指標（分布）として出す。周回の中の経路は、周ごとに比べる |
| foreach | 要素の順序は無視し、要素ごとに比べる |
| parallel | 束の中の順序は無視する |
| decide | 行き先の分布を出す |

全体の指標:

| 指標 | 定義 |
| --- | --- |
| 合格率 | eval × モデルごとの、N回中の合格回数 / N |
| flaky | 合格率が 0 より大きく 1 より小さい eval × モデル |
| route 一致率 | 最頻の route を選んだ実行の割合 |
| 経路一致率 | 上の比べ方で正規化した経路が、最頻の経路と一致した実行の割合 |
| 最初の分岐点 | 最頻の経路から最初に外れた箇所（step・周回・要素）と、その回数 |
| 手順ごとのツール一致率 | step ごとに、呼んだツール集合が最頻の集合と一致した実行の割合 |
| 前回との差 | 保存済みの前回結果（または `--baseline <旧版の skill.yaml>`）との、上記すべての差 |

手順の経路に意味が無いスキル（1回の書き込みで作業が終わる知識型など）は、経路の指標を出さず、
`file_check` と `rule_holds` の合格率で再現性を測る。steps が1つだけのスキルは、自動でこの扱いにする。

### 4.6 レポート

```text
managing-tasks  eval: add-unknown-project  N=5

              合格率   route一致  経路一致  最初の分岐
  haiku        2/5      100%      40%     confirm-project（3回: 確認せずに write-task へ）
  sonnet       5/5      100%     100%     -
  opus         5/5      100%     100%     -
```

- `--format json` で機械可読な結果を出す。GitHub Action はこれを PR の注釈に変換する
- 結果は `.skillrepro/runs/<実行ID>/` に保存し、次回の「前回との差」に使う

### 4.7 コスト上限

- `run` は実行前に「eval 数 × モデル数 × N回 = 合計回数」と推定コストを表示する
- 推定がしきい値（設定 `budget.usd`、既定 $5）を超える場合は、`--yes` が無ければ実行しない
- 実行中に実測コストがしきい値を超えたら、残りの実行を止めてそこまでの結果を出す
- `--live` のときは、スタブにしない `bills` の能力を実行前に一覧で出し、`--yes` が無ければ実行しない

### 4.8 重いスキルの測り方

subagent を多数使うオーケストレーション型のスキルは、1回の実行が長く、N回 × 3モデルは現実的でない。次の3段で測る。

1. **制御だけを測る**: すべての `dispatch` を `stubs.agents` で固定出力にし、orchestrator の制御（周回の上限、差し戻し先、並列発火、gate の位置）だけを N回 × 3モデルで測る
2. **subagent を単独で測る**: 各 subagent を独立したスキルとして skill.yaml にし、入力を fixture にして測る
3. **実モデルでの end-to-end**: 軽量な設定で N=1〜2、手動で回す

## 5. 前提条件の検査

振る舞いを測る前に、環境が揃っていることを確かめる。
揃っていない環境で測ると、ブレの原因が環境なのかスキルなのか切り分けられないため。
`run` は開始前に `lint` と `doctor` を内部で実行し、エラーがあれば測定に進まない。

### 5.1 lint（静的検査）

1ルール = 1ファイルで実装する。各ルールは「どこで・何が・どう直すか」を出す。

Agent Skills とベストプラクティスの検査（生成後の SKILL.md に対して行う）:

- `name`: 64文字以内、小文字・数字・ハイフンのみ、XML タグなし、`anthropic` / `claude` を含まない
- `description`: 空でない、1,024文字以内、XML タグなし、「何をするか」と「いつ使うか」の両方を含む、一人称・二人称で書かない
- 本文が500行未満（超える場合は `knowledge` の `placement: file` を勧める）
- 参照ファイルは SKILL.md から1階層まで。100行を超える参照ファイルには先頭に目次がある
- バックスラッシュのパスがない
- MCP ツールが出力先の形式の完全修飾名になっている

YAML 層の検査（skill.yaml に対して行う）:

- すべての節（スキル・route・step・knowledge・rules・evals）に `name` と `description` がある。`name` が規則に合い、スキル内で一意である
- 1つの step が種類キーを2つ以上持っていない
- `routes` と `steps` を同時に持っていない
- `goto` / `on_fail` / `rules` / `reads` / `{{steps.*}}` の参照先が存在する。`goto` と `on_fail` が 3.4 の範囲の中にある
- `loop` に `max` がある
- `freedom: low` の step に `run` か `call` がある
- 未宣言の依存: 本文・`run`・`call` に、`requires` に無い CLI・MCP 名・環境変数・パス・agent・ルールが出てこない。`/Users/` や `~/` で始まる固定パス、`[[...]]` 形式のリンクがない
- 同梱漏れ: 参照しているファイルが `requires.files` にあり、実在する
- `writes` / `sends` / `bills` の能力を `uses` に持つ step に `gate` がある（`gate: none` のときは `why` がある）
- `evals` が3件以上ある（3件未満は警告）
- 各 eval で、`permissions` の能力と `dispatch` の agent すべてにスタブがある（無ければ警告。`run` では 4.2 の規則で止まる）
- 各 eval で、到達しうる `ask` すべてに `replies` がある（無ければ警告）

意味の判断が要るもの（簡潔さ、用語の一貫性、時間依存の記述、選択肢の多さ）は lint で合否を出さない。
`review` コマンドが観点のチェックリストを Markdown で出力するだけにする。

### 5.2 doctor（環境検査）

`requires` がこの環境で満たせるかを確かめる。

- `tools`: CLI が PATH にあるか
- `mcp`: 割り当て先のツールが接続済みか（既定値を使っている場合はそう表示する）
- `env`: 設定されているか。値は表示せず、設定の有無だけ出す
- `paths`: 存在するか
- `values` / `data`: local に値があるか
- `agents`: 定義ファイルがあるか
- `rules`: ルールファイルがあるか
- `services` / `auth` / `resources`: `check` コマンドが成功するか、出力が `expect` に一致するか
- 足りない項目には「何をすれば直るか」を添える

## 6. compile（SKILL.md の生成）

- `--target claude-code | claude-api | codex` で出力先を選ぶ。能力名はそれぞれの形式のツール名に変換する（Claude Code は `mcp__server__tool`、Claude API は `Server:tool`）
- `--trace` を付けると観測用（4.3 の目印の指示入り）を出す。付けないと本番用
- 出力の先頭に「skill.yaml から生成。手編集禁止」のコメントを入れる
- 制御構造は、読める文章の手順に展開する
  - `routes`: 冒頭に「どのモードかを判定する」節を置き、各 route を見出しにする。各見出しの下に、その route の description と手順を置く
  - `steps`: 番号つきの手順と、進捗チェックリストにする（公式のワークフローパターン）
  - `when` / `optional`: 「〜のときだけ」「（任意）」を手順に添える
  - `loop`: 「最大 N 回まで繰り返す。〜になったら抜ける。N 回で抜けられなければ〜」
  - `foreach`: 「〜のそれぞれについて、次を行う」
  - `parallel`: 「次をすべて同時に始める。1つの完了を待たずに次を始めない」
  - `decide`: 条件と行き先の表
  - `gate`: 「実行前に承認を取る」。条件つき gate は条件と対応の表
- `knowledge` は `placement` に従い、本文に展開するか同梱ファイルにして SKILL.md から1階層で参照する
- `rules` は本文の冒頭近くに「守ること」として並べる

## 7. CLI

| コマンド | 役割 |
| --- | --- |
| `init` | skill.yaml と skill.local.yaml の雛形を作る。`.gitignore` への追記は確認してから行う |
| `lint [path]` | 5.1 の検査 |
| `doctor [path]` | 5.2 の検査 |
| `compile [path] --target <t> [--trace]` | 6 の生成 |
| `run [path] [--models ...] [-n N] [--baseline <path>] [--live]` | 4 の測定 |
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
│   ├── resolve/              # skill.local.yaml との合成、{{...}} の解決
│   ├── rules/                # lint ルール（1ルール = 1ファイル）
│   ├── targets/              # compile の出力先 adapter と、制御構造の文章化
│   ├── doctor/
│   ├── runner/               # Agent SDK 実行・スタブ・一時ディレクトリ・応答
│   ├── trace/                # 目印とツールログの対応づけ
│   ├── metrics/              # 4.5 の指標
│   ├── report/
│   ├── import/
│   └── cli.ts
├── examples/                 # lint を通り、run できる見本スキル
└── action.yml                # GitHub Action
```

examples には、構造の種類ごとに1本ずつ置く: 直列（steps のみ）/ 分岐（routes）/ 対話（ask）/ 繰り返しと委譲（loop・dispatch・parallel）/ 知識型（knowledge・rules・file_check）。

## 9. 実装の順序

中心機能に早く到達する順にする。

1. スキーマと parse、resolve、`compile`（本番用・観測用。制御構造の文章化を含む）
2. `run` / trace / 指標 / レポート（中心機能）。この時点では lint と doctor は最小限（スキーマ検査のみ）
3. lint の全ルール
4. doctor
5. import / review / GitHub Action

## 10. テスト方針

- parse: 構造の種類ごとに「読める YAML」と「読めない YAML」の fixture を置く
- compile: examples の出力をスナップショットで固定する。制御構造の文章化は種類ごとにスナップショットを持つ
- trace と指標: 記録済みのツールログと目印（fixture）から、経路と指標を計算して期待値と比較する。LLM は呼ばない。loop・foreach・parallel の正規化は個別にテストする
- runner: スタブ・一時ディレクトリ・応答（replies / gate）を、本物の SDK を呼ばない偽のエージェントで検証する。スタブ未定義のやり取りと、replies に無い ask で必ず止まることを確認する
- lint: ルールごとに「通る YAML」と「落ちる YAML」の fixture を置き、期待するエラーをスナップショットで比較する
- 移植性の回帰: 固定パス・MCP 名の直書き・同梱漏れ・wikilink など、実在するスキルで見つかった依存パターンの最小例を fixture にし、すべて検出できることを確認する
- 実モデルを使う end-to-end は examples に対してだけ、手動または定期実行で回す（CI の毎回の実行には入れない）

## 11. v0.1 でやらないこと

- Claude 以外のエージェント（Codex CLI など）での実行。compile の出力先としては持つが、`run` は Claude Agent SDK のみ
- 能力名の標準語彙
- スキルの自動修正
- 自然言語の条件（`when` / `decide` の `description`）をツール側で評価すること。評価はエージェントが行い、ツールは目印で観測するだけにする
