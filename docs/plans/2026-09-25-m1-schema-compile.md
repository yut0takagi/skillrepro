# M1: スキーマ・parse・resolve・compile 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** skill.yaml を読み込んで検証し、skill.local.yaml と合成して、Claude Code / Claude API 向けの SKILL.md（本番用・観測用）を生成する CLI `skillrepro compile` を作る。

**Architecture:** `parse` が YAML を行番号つきの型付きモデルに変換して構造エラーを出す。`resolve` が local の値で `{{...}}` を解決し、MCP 能力をツール名に割り当てる。`targets` が制御構造を読める文章の手順に展開して SKILL.md を組み立てる。各段は `Diagnostic` のリストを返し、CLI がエラーの有無で終了コードを決める。

**Tech Stack:** Node.js 24, TypeScript 5.9, `yaml` 2.x, vitest 5, tsx（開発時の実行）

**仕様:** [docs/specs/2026-09-25-skillrepro-design.md](../specs/2026-09-25-skillrepro-design.md) の 3章・6章・9章の段階1

**この計画での判断（仕様との差分）:**

- `compile --target codex` は M1 では実装しない。Codex の MCP ツール名の形式を確認できていないため。指定されたらエラーにする
- `{{env.X}}` は値を埋め込まず、`$X` と出力する。生成した SKILL.md に秘密の値が入らないようにするため
- `{{steps.x.outputs.y}}` は resolve では解決せず、文章化のときに「手順「x」の出力 y」と書く（実行時にしか決まらない値のため）
- `loop.on_exhausted` は step として書く（すべての節を name / description で書く原則に合わせる）。仕様書の例もこれに合わせて直す
- `goto` / `on_fail` の範囲の制限（同じ route・同じ loop の中）と lint の残りのルールは M3 で扱う。M1 は参照先が存在するかだけを見る

---

## ファイル構成

| ファイル | 責務 |
| --- | --- |
| `LICENSE` / `README.md` / `.gitignore` | リポジトリの体裁（Task 1） |
| `package.json` / `tsconfig.json` / `vitest.config.ts` | ビルドとテストの設定 |
| `src/model.ts` | 型定義（Skill / Route / Step / Diagnostic など）。ロジックを持たない |
| `src/parse/yaml-lines.ts` | YAML を行番号つきのプレーンなオブジェクトに変換する |
| `src/parse/parse.ts` | プレーンなオブジェクトを `Skill` に変換し、構造エラーを出す |
| `src/resolve/resolve.ts` | skill.local.yaml の読み込み、`{{...}}` の解決、MCP 能力の割り当て |
| `src/targets/tools.ts` | 出力先ごとのツール名の形式 |
| `src/targets/render-steps.ts` | step と制御構造を文章の手順に展開する |
| `src/targets/compile.ts` | SKILL.md 全体を組み立てる |
| `src/diagnostics.ts` | Diagnostic の表示 |
| `src/cli.ts` | `skillrepro compile` |
| `examples/weekly-digest/` | 見本スキル（直列・gate・call） |
| `examples/managing-tasks/` | 見本スキル（routes・ask・knowledge・rules） |
| `tests/**` | 各モジュールのテスト |

---

### Task 1: リポジトリの体裁（LICENSE・README・.gitignore・作者名・仕様書の例）

**Files:**
- Create: `LICENSE`
- Create: `README.md`
- Create: `.gitignore`
- Modify: `docs/specs/2026-09-25-skillrepro-design.md`（3.4 の `loop` の説明と、`on_exhausted` の例）

- [ ] **Step 1: このリポジトリの作者名を直す**

git の全体設定は変えず、このリポジトリだけに設定する。

```bash
git config --local user.name "Yuto Takagi"
git config --local user.name
```

Expected: `Yuto Takagi`

- [ ] **Step 2: LICENSE を作る**

```text
MIT License

Copyright (c) 2026 Yuto Takagi

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 3: .gitignore を作る**

```gitignore
node_modules/
dist/
.skillrepro/
skill.local.yaml
```

- [ ] **Step 4: README.md を作る**

````markdown
# skillrepro

**Does your agent skill behave the same way every time?**

skillrepro checks the *behavioral reproducibility* of agent skills (Claude Code / Agent Skills).
It runs a skill many times across models and tells you **which step** the runs diverged at —
not just whether the final output passed.

To make that possible, skills are written as structured YAML (`skill.yaml`) and compiled to `SKILL.md`.
Structure lives only where review and observation need it; everything inside is plain natural language.

> Status: design stage. The specification is in [docs/specs](docs/specs/2026-09-25-skillrepro-design.md) (Japanese).

## Why

Skills get copied and shared, and then they quietly stop working:

- **Behavior drifts** — the same request skips a step on one run, or takes a different route on a smaller model.
- **Environments differ** — the skill silently depends on the author's paths, CLIs, MCP servers, env vars, and subagent definitions.

Static linters check `SKILL.md` without running it. Eval tools report pass rates, but not where the runs split.

## What it looks like

```yaml
spec: skillrepro/v0.1
name: weekly-digest
description: |
  Summarizes this week's meeting notes and posts them to Slack.
  Use when: the user asks for a weekly digest of meetings.
steps:
  - name: collect
    description: Collect this week's meeting notes.
    run: python3 scripts/list_notes.py --dir {{paths.notes_dir}} --days 7
    freedom: low
  - name: summarize
    description: Extract decisions and action items, citing the source note for each.
  - name: post
    description: Post the summary to Slack.
    call: { capability: slack.post, args: { channel: "$SLACK_CHANNEL" } }
    gate: confirm
```

```text
weekly-digest  eval: normal-week  N=5
              pass   path-agreement  first divergence
  haiku       2/5         40%        summarize (3 runs: skipped verify, never reached post)
  sonnet      5/5        100%        -
```

## Planned commands

| Command | Purpose |
| --- | --- |
| `skillrepro compile` | Generate `SKILL.md` (production or trace mode) from `skill.yaml` |
| `skillrepro run` | Run evals × models × N times with all external calls stubbed, and report divergence per step |
| `skillrepro lint` | Static checks, including undeclared environment dependencies |
| `skillrepro doctor` | Check whether this machine satisfies the skill's declared requirements |

## 日本語

エージェントのスキルが「何回実行しても、どのモデルでも、同じように振る舞うか」を検査するツールです。
スキルを YAML で手順ごとに構造化して書き、実行のブレを手順単位で特定します。
設計は [仕様書](docs/specs/2026-09-25-skillrepro-design.md) を参照してください。

## License

MIT
````

- [ ] **Step 5: 仕様書の loop の例を、on_exhausted を step として書く形に直す**

`docs/specs/2026-09-25-skillrepro-design.md` の 3.4 の表で、`loop` の行を次に置き換える。

```markdown
| `loop` | 繰り返す。`{max, until, steps, on_exhausted}`。`on_exhausted` は抜けられなかったときに行う step（name / description を持つ） |
```

- [ ] **Step 6: 変更を確認する**

```bash
git status --short
```

Expected（この4ファイルだけ）:

```text
 M docs/specs/2026-09-25-skillrepro-design.md
?? .gitignore
?? LICENSE
?? README.md
```

- [ ] **Step 7: コミットする（ユーザーの承認を取ってから）**

```bash
git add LICENSE README.md .gitignore docs/specs/2026-09-25-skillrepro-design.md
git commit -m "docs: LICENSE・README・.gitignore を追加し、loop の on_exhausted を step に統一"
```

---

### Task 2: プロジェクトの足場

**Files:**
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `vitest.config.ts`
- Test: `tests/smoke.test.ts`

- [ ] **Step 1: package.json を作る**

```json
{
  "name": "skillrepro",
  "version": "0.0.0",
  "description": "Measure whether agent skills behave reproducibly, step by step, across runs and models.",
  "license": "MIT",
  "type": "module",
  "bin": { "skillrepro": "dist/cli.js" },
  "files": ["dist"],
  "engines": { "node": ">=22" },
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "typecheck": "tsc -p tsconfig.json --noEmit",
    "test": "vitest run",
    "dev": "tsx src/cli.ts"
  }
}
```

- [ ] **Step 2: 依存を入れる**

```bash
npm install yaml@^2.9.1
npm install -D typescript@~5.9.3 vitest@^5.0.1 tsx@^4.23.15 @types/node@^24
```

Expected: `package-lock.json` ができ、エラーなく終わる

- [ ] **Step 3: tsconfig.json を作る**

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "outDir": "dist",
    "rootDir": "src",
    "declaration": false,
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

- [ ] **Step 4: vitest.config.ts を作る**

```ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: { include: ['tests/**/*.test.ts'] },
});
```

- [ ] **Step 5: 失敗するスモークテストを書く**

`tests/smoke.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { VERSION } from '../src/model.js';

describe('smoke', () => {
  it('exposes the spec version', () => {
    expect(VERSION).toBe('skillrepro/v0.1');
  });
});
```

- [ ] **Step 6: 失敗を確認する**

Run: `npx vitest run tests/smoke.test.ts`
Expected: FAIL（`../src/model.js` が見つからない）

- [ ] **Step 7: 最小の model.ts を作る**

`src/model.ts`:

```ts
export const VERSION = 'skillrepro/v0.1';
```

- [ ] **Step 8: 通ることを確認する**

Run: `npx vitest run tests/smoke.test.ts && npm run typecheck`
Expected: PASS、型エラーなし

- [ ] **Step 9: コミットする（ユーザーの承認を取ってから）**

```bash
git add package.json package-lock.json tsconfig.json vitest.config.ts src/model.ts tests/smoke.test.ts
git commit -m "chore: TypeScript・vitest の足場を追加"
```

---

### Task 3: 型定義

**Files:**
- Modify: `src/model.ts`

型だけのファイルなのでテストは書かない。Task 4 以降のテストが型を使う。

- [ ] **Step 1: model.ts を型定義で置き換える**

`src/model.ts`:

```ts
export const VERSION = 'skillrepro/v0.1';

export type Severity = 'error' | 'warning';

export interface Diagnostic {
  file: string;
  line: number;
  severity: Severity;
  rule: string;
  message: string;
  fix?: string;
}

export type Freedom = 'high' | 'medium' | 'low';
export type GateKind = 'confirm' | 'confirm_strict' | 'refuse';
export type Gate = 'confirm' | 'none' | { whenExpr: string; kind: GateKind }[];

export interface OnError {
  then: 'stop' | 'continue' | 'goto';
  goto?: string;
  say?: string;
}

export interface DecideCase {
  description: string;
  goto: string;
}

export type StepKind =
  | { type: 'do' }
  | { type: 'run'; command: string }
  | { type: 'call'; capability: string; args: Record<string, unknown> }
  | { type: 'ask'; question: string; choices?: string[] | string }
  | { type: 'dispatch'; agent: string; inputs: Record<string, string>; outputs: Record<string, string> }
  | { type: 'loop'; max: number; until: string; steps: Step[]; onExhausted?: Step }
  | { type: 'foreach'; over: string; as: string; steps: Step[] }
  | { type: 'parallel'; steps: Step[] }
  | { type: 'decide'; cases: DecideCase[] };

export interface Step {
  name: string;
  description: string;
  line: number;
  kind: StepKind;
  when?: string;
  whenExpr?: string;
  optional: boolean;
  freedom?: Freedom;
  why?: string;
  reads: string[];
  rules: string[];
  outputs: Record<string, string>;
  verify?: string;
  onFail?: string;
  onError?: OnError;
  uses: string[];
  gate?: Gate;
}

export interface Route {
  name: string;
  description: string;
  line: number;
  steps: Step[];
}

export interface Named {
  name: string;
  description: string;
  line: number;
}

export interface Knowledge extends Named {
  placement: 'inline' | 'file';
  content: string;
}

export interface Rule extends Named {
  check?: string;
}

export interface McpRequirement {
  description: string;
  default?: string;
}

export interface Requires {
  tools: Record<string, string>;
  mcp: Record<string, McpRequirement>;
  env: Record<string, string>;
  paths: Record<string, string>;
  values: Record<string, string>;
}

export interface Skill {
  file: string;
  spec: string;
  name: string;
  description: string;
  line: number;
  title?: string;
  invokeAs?: string;
  allowedTools: string[];
  purpose?: string;
  inputs: Named[];
  requires: Requires;
  knowledge: Knowledge[];
  rules: Rule[];
  routes?: Route[];
  steps?: Step[];
  doneWhen?: string;
  /** M1 で型に落とさないキー（permissions / evals など）。M2 以降で使う */
  raw: Record<string, unknown>;
}
```

- [ ] **Step 2: 型検査が通ることを確認する**

Run: `npm run typecheck && npx vitest run`
Expected: 型エラーなし、smoke が PASS

- [ ] **Step 3: コミットする（ユーザーの承認を取ってから）**

```bash
git add src/model.ts
git commit -m "feat(model): skill.yaml の型定義を追加"
```

---

### Task 4: YAML を行番号つきで読む

**Files:**
- Create: `src/parse/yaml-lines.ts`
- Test: `tests/parse/yaml-lines.test.ts`

- [ ] **Step 1: 失敗するテストを書く**

`tests/parse/yaml-lines.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { lineOf, loadYaml } from '../../src/parse/yaml-lines.js';

describe('loadYaml', () => {
  it('returns plain objects and remembers the line of each map', () => {
    const { value, errors } = loadYaml('name: a\nsteps:\n  - name: b\n    description: c\n');
    expect(errors).toEqual([]);
    const root = value as { steps: object[] };
    expect(lineOf(root)).toBe(1);
    expect(lineOf(root.steps[0]!)).toBe(3);
    expect(root).toEqual({ name: 'a', steps: [{ name: 'b', description: 'c' }] });
  });

  it('reports syntax errors with a line number', () => {
    const { errors } = loadYaml('name: a\n  bad: [\n');
    expect(errors.length).toBeGreaterThan(0);
    expect(errors[0]!.line).toBeGreaterThanOrEqual(1);
  });
});
```

- [ ] **Step 2: 失敗を確認する**

Run: `npx vitest run tests/parse/yaml-lines.test.ts`
Expected: FAIL（モジュールが見つからない）

- [ ] **Step 3: 実装する**

`src/parse/yaml-lines.ts`:

```ts
import { LineCounter, isMap, isScalar, isSeq, parseDocument } from 'yaml';

const LINES = new WeakMap<object, number>();

export function lineOf(value: unknown): number {
  return typeof value === 'object' && value !== null ? (LINES.get(value) ?? 1) : 1;
}

function toPlain(node: unknown, lc: LineCounter): unknown {
  if (isMap(node)) {
    const obj: Record<string, unknown> = {};
    for (const pair of node.items) {
      const key = isScalar(pair.key) ? String(pair.key.value) : String(pair.key);
      obj[key] = toPlain(pair.value, lc);
    }
    if (node.range) LINES.set(obj, lc.linePos(node.range[0]).line);
    return obj;
  }
  if (isSeq(node)) return node.items.map((item) => toPlain(item, lc));
  if (isScalar(node)) return node.value;
  return node ?? undefined;
}

export interface YamlError {
  line: number;
  message: string;
}

export function loadYaml(text: string): { value: unknown; errors: YamlError[] } {
  const lc = new LineCounter();
  const doc = parseDocument(text, { lineCounter: lc });
  const errors = doc.errors.map((e) => ({ line: e.linePos?.[0]?.line ?? 1, message: e.message }));
  return { value: toPlain(doc.contents, lc), errors };
}
```

- [ ] **Step 4: 通ることを確認する**

Run: `npx vitest run tests/parse/yaml-lines.test.ts`
Expected: PASS

- [ ] **Step 5: コミットする（ユーザーの承認を取ってから）**

```bash
git add src/parse/yaml-lines.ts tests/parse/yaml-lines.test.ts
git commit -m "feat(parse): YAML を行番号つきのプレーンなオブジェクトに変換"
```

---

### Task 5: skill.yaml を Skill に変換する

**Files:**
- Create: `src/parse/parse.ts`
- Test: `tests/parse/parse.test.ts`

- [ ] **Step 1: 失敗するテストを書く**

`tests/parse/parse.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { parseSkill } from '../../src/parse/parse.js';

const HEAD = 'spec: skillrepro/v0.1\nname: demo\ndescription: Demo skill. Use when testing.\n';
const rules = (text: string) => parseSkill(text, 'skill.yaml').diagnostics.map((d) => d.rule);

describe('parseSkill', () => {
  it('parses a sequential skill', () => {
    const { skill, diagnostics } = parseSkill(
      HEAD +
        'steps:\n' +
        '  - name: collect\n    description: Collect notes.\n    run: ls\n    freedom: low\n' +
        '  - name: post\n    description: Post it.\n    call: { capability: slack.post, args: { text: hi } }\n    gate: confirm\n',
      'skill.yaml',
    );
    expect(diagnostics).toEqual([]);
    expect(skill.steps?.map((s) => s.kind.type)).toEqual(['run', 'call']);
    expect(skill.steps?.[1]?.gate).toBe('confirm');
    expect(skill.steps?.[0]?.line).toBe(5);
  });

  it('parses routes, loop, parallel and decide', () => {
    const { skill, diagnostics } = parseSkill(
      HEAD +
        'routes:\n' +
        '  - name: add\n    description: Add a task. Use when adding.\n    steps:\n' +
        '      - name: review\n        description: Review until it passes.\n' +
        '        loop:\n          max: 3\n          until: all reviews pass\n          steps:\n' +
        '            - name: reviewers\n              description: Run reviewers.\n              parallel:\n' +
        '                - { name: logic, description: Logic review., dispatch: { agent: logic-reviewer } }\n' +
        '            - name: rewind\n              description: Choose where to go back.\n              decide:\n' +
        '                cases:\n                  - { description: logic issue, goto: reviewers }\n' +
        '          on_exhausted: { name: give-up, description: Ask what to do., ask: { question: Continue?, choices: [yes, no] } }\n',
      'skill.yaml',
    );
    expect(diagnostics).toEqual([]);
    const loop = skill.routes?.[0]?.steps[0]?.kind;
    expect(loop?.type).toBe('loop');
    if (loop?.type !== 'loop') throw new Error('not a loop');
    expect(loop.max).toBe(3);
    expect(loop.steps.map((s) => s.kind.type)).toEqual(['parallel', 'decide']);
    expect(loop.onExhausted?.kind.type).toBe('ask');
  });

  it('reports missing name and description with the line', () => {
    const { diagnostics } = parseSkill(HEAD + 'steps:\n  - run: ls\n', 'skill.yaml');
    expect(diagnostics.map((d) => [d.rule, d.line])).toEqual([
      ['missing-name', 5],
      ['missing-description', 5],
    ]);
  });

  it('rejects invalid names, duplicates and multiple kind keys', () => {
    expect(rules(HEAD + 'steps:\n  - { name: Bad_Name, description: x }\n')).toEqual(['invalid-name']);
    expect(
      rules(HEAD + 'steps:\n  - { name: a, description: x }\n  - { name: a, description: y }\n'),
    ).toEqual(['duplicate-name']);
    expect(rules(HEAD + 'steps:\n  - { name: a, description: x, run: ls, call: { capability: c } }\n')).toEqual([
      'multiple-kinds',
    ]);
  });

  it('rejects structural mistakes', () => {
    expect(rules(HEAD)).toEqual(['no-steps']);
    expect(
      rules(HEAD + 'steps:\n  - { name: a, description: x }\nroutes:\n  - { name: r, description: y, steps: [] }\n'),
    ).toEqual(['routes-and-steps']);
    expect(rules(HEAD + 'steps:\n  - { name: a, description: x, loop: { until: done, steps: [] } }\n')).toEqual([
      'loop-without-max',
    ]);
    expect(rules(HEAD + 'steps:\n  - { name: a, description: x, freedom: low }\n')).toEqual([
      'low-freedom-without-command',
    ]);
    expect(rules(HEAD + 'steps:\n  - { name: a, description: x, on_fail: nowhere }\n')).toEqual(['unknown-step']);
    expect(rules('spec: other/v9\nname: demo\ndescription: d\nsteps:\n  - { name: a, description: x }\n')).toEqual([
      'unsupported-spec',
    ]);
  });

  it('parses requires, knowledge and rules', () => {
    const { skill, diagnostics } = parseSkill(
      HEAD +
        'requires:\n  paths:\n    - notes_dir: "Where notes live"\n  mcp:\n    - slack.post: { description: Post, default: mcp__slack__post }\n' +
        'knowledge:\n  - { name: projects, description: Project list, content: "a, b" }\n' +
        'rules:\n  - { name: no-overwrite, description: Never overwrite. }\n' +
        'steps:\n  - { name: a, description: x, rules: [no-overwrite] }\n',
      'skill.yaml',
    );
    expect(diagnostics).toEqual([]);
    expect(skill.requires.paths).toEqual({ notes_dir: 'Where notes live' });
    expect(skill.requires.mcp['slack.post']).toEqual({ description: 'Post', default: 'mcp__slack__post' });
    expect(skill.knowledge[0]?.placement).toBe('inline');
    expect(skill.rules[0]?.name).toBe('no-overwrite');
  });
});
```

- [ ] **Step 2: 失敗を確認する**

Run: `npx vitest run tests/parse/parse.test.ts`
Expected: FAIL（モジュールが見つからない）

- [ ] **Step 3: 実装する**

`src/parse/parse.ts`:

```ts
import {
  VERSION,
  type Diagnostic,
  type Gate,
  type GateKind,
  type Knowledge,
  type McpRequirement,
  type Named,
  type Requires,
  type Route,
  type Rule,
  type Skill,
  type Step,
  type StepKind,
} from '../model.js';
import { lineOf, loadYaml } from './yaml-lines.js';

type Obj = Record<string, unknown>;

const NAME = /^[a-z0-9]+(-[a-z0-9]+)*$/;
const KIND_KEYS = ['run', 'call', 'ask', 'dispatch', 'loop', 'foreach', 'parallel', 'decide'] as const;
const GATE_KINDS: GateKind[] = ['confirm', 'confirm_strict', 'refuse'];

const isObj = (v: unknown): v is Obj => typeof v === 'object' && v !== null && !Array.isArray(v);
const str = (v: unknown): string | undefined => (typeof v === 'string' ? v : undefined);
const strList = (v: unknown): string[] => (Array.isArray(v) ? v.filter((x): x is string => typeof x === 'string') : []);
const strMap = (v: unknown): Record<string, string> =>
  isObj(v) ? Object.fromEntries(Object.entries(v).map(([k, x]) => [k, String(x ?? '')])) : {};

/** `- key: value` のリスト（または map）を Record にする */
function entries(v: unknown): Record<string, unknown> {
  if (isObj(v)) return v;
  const out: Record<string, unknown> = {};
  if (Array.isArray(v)) for (const item of v) if (isObj(item)) Object.assign(out, item);
  return out;
}

class Ctx {
  diagnostics: Diagnostic[] = [];
  constructor(readonly file: string) {}
  error(line: number, rule: string, message: string, fix?: string) {
    this.diagnostics.push({ file: this.file, line, severity: 'error', rule, message, fix });
  }
}

function named(raw: Obj, line: number, ctx: Ctx, what: string): Named {
  const name = str(raw.name);
  const description = str(raw.description);
  if (name === undefined) {
    ctx.error(line, 'missing-name', `${what}に name がありません。`, 'name: <小文字・数字・ハイフン> を追加してください。');
  } else if (!NAME.test(name) || name.length > 64) {
    ctx.error(line, 'invalid-name', `name「${name}」が規則に合いません。`, '64文字以内の小文字・数字・ハイフンにしてください（例: collect-notes）。');
  }
  if (description === undefined || description.trim() === '') {
    ctx.error(line, 'missing-description', `${what}「${name ?? '?'}」に description がありません。`, 'description: に何をするかを文章で書いてください。');
  }
  return { name: name ?? '', description: description ?? '', line };
}

function parseSteps(v: unknown, ctx: Ctx): Step[] {
  if (!Array.isArray(v)) return [];
  return v.map((item) => parseStep(item, ctx)).filter((s): s is Step => s !== undefined);
}

function parseGate(v: unknown, line: number, ctx: Ctx): Gate | undefined {
  if (v === undefined) return undefined;
  if (v === 'confirm' || v === 'none') return v;
  if (Array.isArray(v)) {
    const rules = v.filter(isObj).map((r) => ({ whenExpr: str(r.when_expr) ?? '', kind: str(r.kind) as GateKind }));
    if (rules.every((r) => r.whenExpr !== '' && GATE_KINDS.includes(r.kind))) return rules;
  }
  ctx.error(line, 'invalid-gate', 'gate の書き方が正しくありません。', 'confirm / none、または { when_expr, kind } のリストにしてください。');
  return undefined;
}

function parseKind(key: string | undefined, raw: Obj, line: number, ctx: Ctx): StepKind {
  const v = key === undefined ? undefined : raw[key];
  switch (key) {
    case undefined:
      return { type: 'do' };
    case 'run':
      return { type: 'run', command: str(v) ?? '' };
    case 'call': {
      const c = isObj(v) ? v : {};
      return { type: 'call', capability: str(c.capability) ?? '', args: isObj(c.args) ? c.args : {} };
    }
    case 'ask': {
      const a = isObj(v) ? v : {};
      const choices = Array.isArray(a.choices) ? strList(a.choices) : str(a.choices);
      if (str(a.question) === undefined) ctx.error(line, 'ask-without-question', 'ask に question がありません。', 'ask: { question: ... } と書いてください。');
      return { type: 'ask', question: str(a.question) ?? '', ...(choices !== undefined ? { choices } : {}) };
    }
    case 'dispatch': {
      const d = isObj(v) ? v : {};
      return { type: 'dispatch', agent: str(d.agent) ?? '', inputs: strMap(d.inputs), outputs: strMap(d.outputs) };
    }
    case 'loop': {
      const l = isObj(v) ? v : {};
      if (typeof l.max !== 'number' || l.max < 1) {
        ctx.error(line, 'loop-without-max', 'loop に max（1以上の整数）がありません。', '無限に回らないよう max: 3 のように上限を書いてください。');
      }
      const onExhausted = l.on_exhausted === undefined ? undefined : parseStep(l.on_exhausted, ctx);
      return {
        type: 'loop',
        max: typeof l.max === 'number' ? l.max : 0,
        until: str(l.until) ?? '',
        steps: parseSteps(l.steps, ctx),
        ...(onExhausted ? { onExhausted } : {}),
      };
    }
    case 'foreach': {
      const f = isObj(v) ? v : {};
      return { type: 'foreach', over: str(f.over) ?? '', as: str(f.as) ?? 'item', steps: parseSteps(f.steps, ctx) };
    }
    case 'parallel':
      return { type: 'parallel', steps: parseSteps(v, ctx) };
    case 'decide': {
      const d = isObj(v) ? v : {};
      const cases = (Array.isArray(d.cases) ? d.cases : [])
        .filter(isObj)
        .map((c) => ({ description: str(c.description) ?? '', goto: str(c.goto) ?? '' }));
      return { type: 'decide', cases };
    }
    default:
      return { type: 'do' };
  }
}

function parseStep(raw: unknown, ctx: Ctx): Step | undefined {
  if (!isObj(raw)) {
    ctx.error(lineOf(raw), 'step-not-map', '手順がマップになっていません。', '- name: ... / description: ... の形で書いてください。');
    return undefined;
  }
  const line = lineOf(raw);
  const { name, description } = named(raw, line, ctx, '手順');
  const kinds = KIND_KEYS.filter((k) => k in raw);
  if (kinds.length > 1) {
    ctx.error(line, 'multiple-kinds', `手順「${name}」が種類キーを${kinds.length}つ持っています（${kinds.join(', ')}）。`, '1つの手順に種類キーは1つまでです。手順を分けてください。');
  }
  const kind = parseKind(kinds[0], raw, line, ctx);
  const freedom = str(raw.freedom) as Step['freedom'];
  if (freedom === 'low' && kind.type !== 'run' && kind.type !== 'call') {
    ctx.error(line, 'low-freedom-without-command', `手順「${name}」は freedom: low なのに run も call もありません。`, '正確に実行させたいコマンドを run か call に書いてください。');
  }
  const onError = isObj(raw.on_error)
    ? { then: (str(raw.on_error.then) ?? 'stop') as 'stop' | 'continue' | 'goto', goto: str(raw.on_error.goto), say: str(raw.on_error.say) }
    : undefined;
  const gate = parseGate(raw.gate, line, ctx);
  return {
    name,
    description,
    line,
    kind,
    when: str(raw.when),
    whenExpr: str(raw.when_expr),
    optional: raw.optional === true,
    freedom,
    why: str(raw.why),
    reads: strList(raw.reads),
    rules: strList(raw.rules),
    outputs: strMap(raw.outputs),
    verify: str(raw.verify),
    onFail: str(raw.on_fail),
    ...(onError ? { onError } : {}),
    uses: strList(raw.uses),
    ...(gate !== undefined ? { gate } : {}),
  };
}

function parseRequires(v: unknown): Requires {
  const r = isObj(v) ? v : {};
  const mcp: Record<string, McpRequirement> = {};
  for (const [cap, m] of Object.entries(entries(r.mcp))) {
    mcp[cap] = isObj(m)
      ? { description: str(m.description) ?? '', ...(str(m.default) ? { default: str(m.default) } : {}) }
      : { description: String(m ?? '') };
  }
  const flat = (x: unknown) => strMap(entries(x));
  return { tools: flat(r.tools), mcp, env: flat(r.env), paths: flat(r.paths), values: flat(r.values) };
}

/** すべての step（入れ子を含む）を列挙する */
export function walkSteps(steps: Step[], visit: (s: Step) => void): void {
  for (const s of steps) {
    visit(s);
    const k = s.kind;
    if (k.type === 'loop') {
      walkSteps(k.steps, visit);
      if (k.onExhausted) walkSteps([k.onExhausted], visit);
    } else if (k.type === 'foreach' || k.type === 'parallel') {
      walkSteps(k.steps, visit);
    }
  }
}

export function allSteps(skill: Pick<Skill, 'steps' | 'routes'>): Step[] {
  const out: Step[] = [];
  walkSteps(skill.steps ?? [], (s) => out.push(s));
  for (const r of skill.routes ?? []) walkSteps(r.steps, (s) => out.push(s));
  return out;
}

function checkReferences(skill: Skill, ctx: Ctx) {
  const seen = new Map<string, number>();
  const note = (name: string, line: number) => {
    if (name === '') return;
    if (seen.has(name)) {
      ctx.error(line, 'duplicate-name', `name「${name}」が重複しています（${seen.get(name)}行目でも使われています）。`, 'route と step の name はスキルの中で一意にしてください。');
    } else seen.set(name, line);
  };
  for (const r of skill.routes ?? []) note(r.name, r.line);
  const steps = allSteps(skill);
  for (const s of steps) note(s.name, s.line);
  const exists = (n: string) => steps.some((s) => s.name === n);
  for (const s of steps) {
    const targets = [s.onFail, s.onError?.goto, ...(s.kind.type === 'decide' ? s.kind.cases.map((c) => c.goto) : [])];
    for (const t of targets) {
      if (t !== undefined && !exists(t)) {
        ctx.error(s.line, 'unknown-step', `手順「${s.name}」の行き先「${t}」という手順がありません。`, '行き先には存在する手順の name を書いてください。');
      }
    }
  }
}

export function parseSkill(text: string, file: string): { skill: Skill; diagnostics: Diagnostic[] } {
  const ctx = new Ctx(file);
  const { value, errors } = loadYaml(text);
  for (const e of errors) ctx.error(e.line, 'yaml-syntax', `YAML として読めません: ${e.message}`);
  const raw = isObj(value) ? value : {};
  const line = lineOf(raw);

  const spec = str(raw.spec) ?? '';
  if (spec !== VERSION) {
    ctx.error(line, 'unsupported-spec', `spec「${spec}」には対応していません。`, `spec: ${VERSION} と書いてください。`);
  }
  const head = named(raw, line, ctx, 'スキル');

  const namedList = <T extends Named>(v: unknown, what: string, extra: (o: Obj) => Omit<T, keyof Named>): T[] =>
    (Array.isArray(v) ? v : []).filter(isObj).map((o) => ({ ...named(o, lineOf(o), ctx, what), ...extra(o) }) as T);

  const knowledge = namedList<Knowledge>(raw.knowledge, '知識', (o) => ({
    placement: o.placement === 'file' ? 'file' : 'inline',
    content: str(o.content) ?? '',
  }));
  const rules = namedList<Rule>(raw.rules, '規則', (o) => (str(o.check) ? { check: str(o.check) } : {}));
  const inputs = namedList<Named>(raw.inputs, '入力', () => ({}));

  let routes: Route[] | undefined;
  if (Array.isArray(raw.routes)) {
    routes = raw.routes.filter(isObj).map((r) => ({ ...named(r, lineOf(r), ctx, 'route'), steps: parseSteps(r.steps, ctx) }));
  }
  const steps = Array.isArray(raw.steps) ? parseSteps(raw.steps, ctx) : undefined;
  if (routes && steps) ctx.error(line, 'routes-and-steps', 'routes と steps を同時に持っています。', 'どちらか一方にしてください。モード分岐があるなら routes の中に steps を書きます。');
  if (!routes && !steps) ctx.error(line, 'no-steps', 'routes も steps もありません。', '手順を steps に書いてください。');

  const known = new Set(['spec', 'name', 'description', 'title', 'invoke_as', 'allowed-tools', 'purpose', 'inputs', 'requires', 'knowledge', 'rules', 'routes', 'steps', 'done_when']);
  const skill: Skill = {
    file,
    spec,
    ...head,
    title: str(raw.title),
    invokeAs: str(raw.invoke_as),
    allowedTools: strList(raw['allowed-tools']),
    purpose: str(raw.purpose),
    inputs,
    requires: parseRequires(raw.requires),
    knowledge,
    rules,
    ...(routes ? { routes } : {}),
    ...(steps ? { steps } : {}),
    doneWhen: str(raw.done_when),
    raw: Object.fromEntries(Object.entries(raw).filter(([k]) => !known.has(k))),
  };
  checkReferences(skill, ctx);
  return { skill, diagnostics: ctx.diagnostics };
}
```

- [ ] **Step 4: 通ることを確認する**

Run: `npx vitest run tests/parse/parse.test.ts && npm run typecheck`
Expected: PASS（6件）、型エラーなし

- [ ] **Step 5: コミットする（ユーザーの承認を取ってから）**

```bash
git add src/parse/parse.ts tests/parse/parse.test.ts
git commit -m "feat(parse): skill.yaml を Skill モデルに変換し構造エラーを検出"
```

---

### Task 6: skill.local.yaml との合成

**Files:**
- Create: `src/resolve/resolve.ts`
- Test: `tests/resolve/resolve.test.ts`

- [ ] **Step 1: 失敗するテストを書く**

`tests/resolve/resolve.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { parseSkill } from '../../src/parse/parse.js';
import { parseLocal, resolveSkill } from '../../src/resolve/resolve.js';

const skillOf = (body: string) =>
  parseSkill(
    'spec: skillrepro/v0.1\nname: demo\ndescription: Demo. Use when testing.\n' +
      'requires:\n  paths:\n    - notes_dir: notes\n  env:\n    - TOKEN_NAME: channel\n' +
      '  mcp:\n    - slack.post: { description: Post, default: mcp__slack__post }\n    - cal.list: { description: Calendar }\n' +
      body,
    'skill.yaml',
  ).skill;

describe('resolveSkill', () => {
  it('substitutes paths, keeps env as a variable and binds capabilities', () => {
    const skill = skillOf(
      'steps:\n  - { name: a, description: "Read {{paths.notes_dir}} and send to {{env.TOKEN_NAME}}", uses: [slack.post, cal.list] }\n',
    );
    const local = parseLocal('paths:\n  notes_dir: work/notes\nmcp:\n  cal.list: mcp__gcal__list\n');
    const r = resolveSkill(skill, local);
    expect(r.diagnostics).toEqual([]);
    expect(r.skill.steps?.[0]?.description).toBe('Read work/notes and send to $TOKEN_NAME');
    expect(r.tools).toEqual({ 'slack.post': 'mcp__slack__post', 'cal.list': 'mcp__gcal__list' });
    expect(r.notices).toEqual(['能力「slack.post」は既定値 mcp__slack__post を使用中です（skill.local.yaml で割り当てていません）。']);
  });

  it('turns a whole-template string into a list from local data', () => {
    const skill = skillOf('steps:\n  - { name: a, description: x, ask: { question: Which?, choices: "{{local.data.projects}}" } }\n');
    const r = resolveSkill(skill, parseLocal('paths:\n  notes_dir: n\nmcp:\n  cal.list: t\ndata:\n  projects: [p1, p2]\n'));
    const kind = r.skill.steps?.[0]?.kind;
    expect(kind).toEqual({ type: 'ask', question: 'Which?', choices: ['p1', 'p2'] });
  });

  it('leaves step outputs untouched', () => {
    const skill = skillOf('steps:\n  - { name: a, description: "Use {{steps.b.outputs.gb}}" }\n  - { name: b, description: y }\n');
    const r = resolveSkill(skill, parseLocal('paths:\n  notes_dir: n\nmcp:\n  cal.list: t\n'));
    expect(r.skill.steps?.[0]?.description).toBe('Use {{steps.b.outputs.gb}}');
  });

  it('reports missing, undeclared and unbound values', () => {
    const skill = skillOf('steps:\n  - { name: a, description: "{{paths.notes_dir}} {{values.project}} {{foo.bar}}", uses: [nope] }\n');
    const r = resolveSkill(skill, parseLocal(''));
    expect(r.diagnostics.map((d) => d.rule).sort()).toEqual(
      ['missing-local-value', 'undeclared-reference', 'unknown-reference', 'unbound-capability', 'undeclared-capability'].sort(),
    );
    expect(r.diagnostics.find((d) => d.rule === 'missing-local-value')?.line).toBe(13);
  });
});
```

- [ ] **Step 2: 失敗を確認する**

Run: `npx vitest run tests/resolve/resolve.test.ts`
Expected: FAIL（モジュールが見つからない）

- [ ] **Step 3: 実装する**

`src/resolve/resolve.ts`:

```ts
import { parse as parseYaml } from 'yaml';
import type { Diagnostic, Skill } from '../model.js';
import { allSteps } from '../parse/parse.js';

export interface Local {
  paths: Record<string, string>;
  values: Record<string, string>;
  mcp: Record<string, string>;
  env: Record<string, string>;
  data: Record<string, unknown>;
}

export interface Resolved {
  skill: Skill;
  /** 能力名 → ツール名（Claude Code の形式） */
  tools: Record<string, string>;
  diagnostics: Diagnostic[];
  notices: string[];
}

type Obj = Record<string, unknown>;
const isObj = (v: unknown): v is Obj => typeof v === 'object' && v !== null && !Array.isArray(v);
const section = (v: unknown): Record<string, string> =>
  isObj(v) ? Object.fromEntries(Object.entries(v).map(([k, x]) => [k, String(x)])) : {};

export function parseLocal(text: string): Local {
  const raw: unknown = text.trim() === '' ? {} : parseYaml(text);
  const o = isObj(raw) ? raw : {};
  return {
    paths: section(o.paths),
    values: section(o.values),
    mcp: section(o.mcp),
    env: section(o.env),
    data: isObj(o.data) ? o.data : {},
  };
}

const TEMPLATE = /\{\{\s*([^{}\s]+)\s*\}\}/g;
const WHOLE = /^\{\{\s*([^{}\s]+)\s*\}\}$/;

/** 文字列を再帰的に置き換える。最も近い line を持つオブジェクトの行番号を fn に渡す */
function mapStrings(value: unknown, fn: (s: string, line: number) => unknown, line: number): unknown {
  if (typeof value === 'string') return fn(value, line);
  if (Array.isArray(value)) return value.map((v) => mapStrings(v, fn, line));
  if (isObj(value)) {
    const here = typeof value.line === 'number' ? value.line : line;
    return Object.fromEntries(Object.entries(value).map(([k, v]) => [k, k === 'raw' ? v : mapStrings(v, fn, here)]));
  }
  return value;
}

export function resolveSkill(skill: Skill, local: Local): Resolved {
  const diagnostics: Diagnostic[] = [];
  const reported = new Set<string>();
  const error = (line: number, rule: string, message: string, fix: string) => {
    const key = `${rule}:${message}`;
    if (reported.has(key)) return;
    reported.add(key);
    diagnostics.push({ file: skill.file, line, severity: 'error', rule, message, fix });
  };

  const lookup = (ref: string, line: number): unknown => {
    const [ns, key, sub] = ref.split('.');
    if (ns === 'steps') return undefined;
    if (ns === 'paths' || ns === 'values') {
      if (key === undefined || !(key in skill.requires[ns])) {
        error(line, 'undeclared-reference', `{{${ref}}} は requires.${ns} に宣言されていません。`, `requires.${ns} に「- ${key}: <説明>」を追加してください。`);
        return undefined;
      }
      const v = local[ns][key];
      if (v === undefined) error(line, 'missing-local-value', `{{${ref}}} の値が skill.local.yaml にありません。`, `skill.local.yaml の ${ns} に「${key}: <値>」を追加してください。`);
      return v;
    }
    if (ns === 'env') {
      if (key === undefined || !(key in skill.requires.env)) {
        error(line, 'undeclared-reference', `{{${ref}}} は requires.env に宣言されていません。`, `requires.env に「- ${key}: <説明>」を追加してください。`);
        return undefined;
      }
      return `$${key}`;
    }
    if (ns === 'local' && key === 'data' && sub !== undefined) {
      const v = local.data[sub];
      if (v === undefined) error(line, 'missing-local-value', `{{${ref}}} の値が skill.local.yaml にありません。`, `skill.local.yaml の data に「${sub}: <値>」を追加してください。`);
      return v;
    }
    error(line, 'unknown-reference', `{{${ref}}} は参照できません。`, 'paths / values / env / local.data / steps のいずれかを参照してください。');
    return undefined;
  };

  const substitute = (s: string, line: number): unknown => {
    const whole = s.match(WHOLE);
    if (whole?.[1]) {
      const v = lookup(whole[1], line);
      if (Array.isArray(v)) return v;
      return v === undefined ? s : String(v);
    }
    return s.replace(TEMPLATE, (m, ref: string) => {
      const v = lookup(ref, line);
      if (v === undefined) return m;
      return Array.isArray(v) ? v.join(' / ') : String(v);
    });
  };

  const resolved = mapStrings(skill, substitute, skill.line) as Skill;

  const tools: Record<string, string> = {};
  const notices: string[] = [];
  for (const [cap, req] of Object.entries(skill.requires.mcp)) {
    const bound = local.mcp[cap];
    if (bound !== undefined) tools[cap] = bound;
    else if (req.default !== undefined) {
      tools[cap] = req.default;
      notices.push(`能力「${cap}」は既定値 ${req.default} を使用中です（skill.local.yaml で割り当てていません）。`);
    } else {
      error(skill.line, 'unbound-capability', `能力「${cap}」にツールが割り当てられていません。`, `skill.local.yaml の mcp に「${cap}: <ツール名>」を追加してください。`);
    }
  }
  for (const s of allSteps(resolved)) {
    const caps = [...s.uses, ...(s.kind.type === 'call' ? [s.kind.capability] : [])];
    for (const cap of caps) {
      if (!(cap in skill.requires.mcp)) {
        error(s.line, 'undeclared-capability', `手順「${s.name}」の能力「${cap}」が requires.mcp に宣言されていません。`, `requires.mcp に「- ${cap}: { description: <用途> }」を追加してください。`);
      }
    }
  }
  return { skill: resolved, tools, diagnostics, notices };
}
```

- [ ] **Step 4: 通ることを確認する**

Run: `npx vitest run tests/resolve/resolve.test.ts && npm run typecheck`
Expected: PASS（4件）、型エラーなし

- [ ] **Step 5: コミットする（ユーザーの承認を取ってから）**

```bash
git add src/resolve/resolve.ts tests/resolve/resolve.test.ts
git commit -m "feat(resolve): skill.local.yaml との合成と能力の割り当て"
```

---

### Task 7: 出力先ごとのツール名

**Files:**
- Create: `src/targets/tools.ts`
- Test: `tests/targets/tools.test.ts`

- [ ] **Step 1: 失敗するテストを書く**

`tests/targets/tools.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { formatTool, parseTarget } from '../../src/targets/tools.js';

describe('formatTool', () => {
  it('keeps Claude Code names and converts to Server:tool for the API', () => {
    expect(formatTool('mcp__slack__slack_post_message', 'claude-code')).toBe('mcp__slack__slack_post_message');
    expect(formatTool('mcp__slack__slack_post_message', 'claude-api')).toBe('slack:slack_post_message');
    expect(formatTool('Read', 'claude-api')).toBe('Read');
  });
});

describe('parseTarget', () => {
  it('accepts supported targets only', () => {
    expect(parseTarget('claude-code')).toBe('claude-code');
    expect(parseTarget('claude-api')).toBe('claude-api');
    expect(parseTarget('codex')).toBeUndefined();
  });
});
```

- [ ] **Step 2: 失敗を確認する**

Run: `npx vitest run tests/targets/tools.test.ts`
Expected: FAIL（モジュールが見つからない）

- [ ] **Step 3: 実装する**

`src/targets/tools.ts`:

```ts
export type Target = 'claude-code' | 'claude-api';

export const TARGETS: Target[] = ['claude-code', 'claude-api'];

export function parseTarget(value: string): Target | undefined {
  return (TARGETS as string[]).includes(value) ? (value as Target) : undefined;
}

/** Claude Code 形式（mcp__server__tool）のツール名を出力先の形式にする */
export function formatTool(tool: string, target: Target): string {
  if (target === 'claude-code') return tool;
  const m = tool.match(/^mcp__(.+?)__(.+)$/);
  return m ? `${m[1]}:${m[2]}` : tool;
}
```

- [ ] **Step 4: 通ることを確認する**

Run: `npx vitest run tests/targets/tools.test.ts`
Expected: PASS

- [ ] **Step 5: コミットする（ユーザーの承認を取ってから）**

```bash
git add src/targets/tools.ts tests/targets/tools.test.ts
git commit -m "feat(targets): 出力先ごとのツール名の形式"
```

---

### Task 8: 手順と制御構造の文章化

**Files:**
- Create: `src/targets/render-steps.ts`
- Test: `tests/targets/render-steps.test.ts`

- [ ] **Step 1: 失敗するテストを書く**

`tests/targets/render-steps.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { parseSkill } from '../../src/parse/parse.js';
import { renderSteps, type RenderContext } from '../../src/targets/render-steps.js';

const stepsOf = (yaml: string) =>
  parseSkill('spec: skillrepro/v0.1\nname: demo\ndescription: Demo. Use when testing.\nsteps:\n' + yaml, 'skill.yaml').skill.steps ?? [];

const ctx = (trace: boolean): RenderContext => ({ target: 'claude-code', trace, tools: { 'slack.post': 'mcp__slack__post' } });
const render = (yaml: string, trace = false) => renderSteps(stepsOf(yaml), ctx(trace), 3).join('\n');

describe('renderSteps', () => {
  it('renders run with low freedom, when, gate and verify', () => {
    const out = render(
      '  - name: collect\n    description: Collect notes.\n    run: ls notes\n    freedom: low\n    when: notes exist\n' +
        '    verify: every note has a title\n    on_fail: collect\n    gate: confirm\n    why: Keeps it exact.\n',
    );
    expect(out).toContain('### 1. collect');
    expect(out).toContain('Collect notes.');
    expect(out).toContain('**このときだけ行う:** notes exist');
    expect(out).toContain('```bash\nls notes\n```');
    expect(out).toContain('コマンドを変更したり、フラグを足したりしない。');
    expect(out).toContain('**確認:** every note has a title。満たさなければ手順「collect」に戻る。');
    expect(out).toContain('**実行前に、ユーザーの承認を取る。** 承認されなければ実行しない。');
    expect(out).toContain('_理由: Keeps it exact._');
  });

  it('renders call with the resolved tool name and step output references', () => {
    const out = render(
      '  - name: post\n    description: Post {{steps.sum.outputs.text}}.\n    call: { capability: slack.post, args: { text: hi } }\n' +
        '  - name: sum\n    description: Summarize.\n    outputs: { text: the summary }\n',
    );
    expect(out).toContain('Post 手順「sum」の出力 text.');
    expect(out).toContain('`mcp__slack__post` を次の引数で呼ぶ:');
    expect(out).toContain('"text": "hi"');
    expect(out).toContain('**この手順の出力:**\n- text: the summary');
  });

  it('renders loop, parallel, decide and ask', () => {
    const out = render(
      '  - name: review\n    description: Review until it passes.\n    loop:\n      max: 3\n      until: all reviews pass\n      steps:\n' +
        '        - name: reviewers\n          description: Run reviewers.\n          parallel:\n' +
        '            - { name: logic, description: Logic review., dispatch: { agent: logic-reviewer } }\n' +
        '        - name: rewind\n          description: Choose.\n          decide:\n            cases:\n              - { description: logic issue, goto: reviewers }\n' +
        '      on_exhausted: { name: give-up, description: Ask., ask: { question: Continue?, choices: [yes, no] } }\n',
    );
    expect(out).toContain('最大 3 回まで、次の手順を繰り返す。all reviews pass になったら繰り返しを抜ける。');
    expect(out).toContain('#### 1.1. reviewers');
    expect(out).toContain('次の手順をすべて同時に始める。1つの完了を待ってから次を始めることはしない。');
    expect(out).toContain('subagent `logic-reviewer` に委譲する。');
    expect(out).toContain('| logic issue | reviewers |');
    expect(out).toContain('3 回で抜けられなかったときは、次を行う:');
    expect(out).toContain('> Continue?');
    expect(out).toContain('選択肢: yes / no');
  });

  it('adds trace markers only in trace mode', () => {
    const yaml =
      '  - name: a\n    description: A.\n    when: cond\n' +
      '  - name: b\n    description: B.\n    loop: { max: 2, until: done, steps: [{ name: c, description: C., ask: { question: Q? } }] }\n';
    const plain = render(yaml, false);
    expect(plain).not.toContain('[step:');
    const traced = render(yaml, true);
    expect(traced).toContain('最初に `[step:a]` を出力する。');
    expect(traced).toContain('`[skip:a <理由>]` を出力して飛ばす。');
    expect(traced).toContain('各周の始めに `[round:b <n>]` を出力する（n は 1 から数える）。');
    expect(traced).toContain('質問する直前に `[ask:c]` を出力する。');
  });
});
```

- [ ] **Step 2: 失敗を確認する**

Run: `npx vitest run tests/targets/render-steps.test.ts`
Expected: FAIL（モジュールが見つからない）

- [ ] **Step 3: 実装する**

`src/targets/render-steps.ts`:

```ts
import type { Gate, Step } from '../model.js';
import { formatTool, type Target } from './tools.js';

export interface RenderContext {
  target: Target;
  trace: boolean;
  /** 能力名 → ツール名（Claude Code の形式） */
  tools: Record<string, string>;
}

const STEP_REF = /\{\{\s*steps\.([a-z0-9-]+)\.outputs\.([A-Za-z0-9_-]+)\s*\}\}/g;

export function prose(s: string): string {
  return s.replace(STEP_REF, (_m, step: string, key: string) => `手順「${step}」の出力 ${key}`).trim();
}

function toolName(cap: string, ctx: RenderContext): string {
  return formatTool(ctx.tools[cap] ?? cap, ctx.target);
}

function renderGate(step: Step, gate: Gate, ctx: RenderContext): string[] {
  if (gate === 'none') return [];
  const out: string[] = [];
  if (gate === 'confirm') {
    out.push('**実行前に、ユーザーの承認を取る。** 承認されなければ実行しない。');
  } else {
    const label = { confirm: '承認を取る', confirm_strict: '明確な承認を取る（曖昧な返答は承認とみなさない）', refuse: '実行しない' };
    out.push('**実行前に、次の条件で承認を判断する（上から順に、最初に当てはまったもの）:**', '', '| 条件 | 対応 |', '| --- | --- |');
    for (const r of gate) out.push(`| \`${prose(r.whenExpr)}\` | ${label[r.kind]} |`);
    out.push('', 'どれにも当てはまらなければ、承認なしで進む。');
  }
  if (ctx.trace) out.push(`承認を求める直前に \`[gate:${step.name} <種類>]\` を出力する（種類は confirm / confirm_strict / refuse）。`);
  return out;
}

function renderKind(step: Step, num: string, ctx: RenderContext, depth: number): string[] {
  const k = step.kind;
  const low = step.freedom === 'low';
  switch (k.type) {
    case 'do':
      return [];
    case 'run':
      return ['次のコマンドをそのまま実行する:', '', '```bash', k.command, '```', ...(low ? ['', 'コマンドを変更したり、フラグを足したりしない。'] : [])];
    case 'call':
      return [
        `\`${toolName(k.capability, ctx)}\` を次の引数で呼ぶ:`,
        '',
        '```json',
        JSON.stringify(k.args, null, 2),
        '```',
        ...(low ? ['', '引数を変更したり、足したりしない。'] : []),
      ];
    case 'ask': {
      const out = [...(ctx.trace ? [`質問する直前に \`[ask:${step.name}]\` を出力する。`] : []), 'ユーザーに次の質問をし、回答を待つ:', '', `> ${prose(k.question)}`];
      if (Array.isArray(k.choices)) out.push('', `選択肢: ${k.choices.join(' / ')}`);
      else if (k.choices) out.push('', `選択肢: ${prose(k.choices)}`);
      return out;
    }
    case 'dispatch': {
      const out = [...(ctx.trace ? [`委譲する直前に \`[dispatch:${step.name} ${k.agent}]\` を出力する。`] : []), `subagent \`${k.agent}\` に委譲する。`];
      if (Object.keys(k.inputs).length) out.push('', '渡すもの:', ...Object.entries(k.inputs).map(([key, v]) => `- ${key}: ${prose(v)}`));
      if (Object.keys(k.outputs).length) out.push('', '受け取るもの:', ...Object.entries(k.outputs).map(([key, v]) => `- ${key}: ${prose(v)}`));
      return out;
    }
    case 'loop': {
      const out = [
        `最大 ${k.max} 回まで、次の手順を繰り返す。${prose(k.until)} になったら繰り返しを抜ける。`,
        ...(ctx.trace ? [`各周の始めに \`[round:${step.name} <n>]\` を出力する（n は 1 から数える）。`] : []),
        '',
        ...renderStepList(k.steps, ctx, depth + 1, num),
      ];
      if (k.onExhausted) out.push('', `${k.max} 回で抜けられなかったときは、次を行う:`, '', ...renderStep(k.onExhausted, `${num}.${k.steps.length + 1}`, ctx, depth + 1));
      return out;
    }
    case 'foreach':
      return [
        `${prose(k.over)} のそれぞれ（以下「${k.as}」）について、次の手順を行う。`,
        ...(ctx.trace ? [`各要素に入るときに \`[item:${step.name} <要素の名前>]\` を出力する。`] : []),
        '',
        ...renderStepList(k.steps, ctx, depth + 1, num),
      ];
    case 'parallel':
      return ['次の手順をすべて同時に始める。1つの完了を待ってから次を始めることはしない。', '', ...renderStepList(k.steps, ctx, depth + 1, num)];
    case 'decide':
      return [
        '次の表で、次に進む手順を決める。',
        ...(ctx.trace ? [`決めたら \`[decide:${step.name} <進む手順>]\` を出力する。`] : []),
        '',
        '| 条件 | 次に進む手順 |',
        '| --- | --- |',
        ...k.cases.map((c) => `| ${prose(c.description)} | ${c.goto} |`),
      ];
  }
}

export function renderStep(step: Step, num: string, ctx: RenderContext, depth: number): string[] {
  const level = '#'.repeat(Math.min(3 + depth, 6));
  const out = [`${level} ${num}. ${step.name}${step.optional ? '（任意）' : ''}`, ''];
  if (ctx.trace) out.push(`最初に \`[step:${step.name}]\` を出力する。`, '');
  out.push(prose(step.description));
  if (step.when) {
    out.push('', `**このときだけ行う:** ${prose(step.when)}`);
    if (ctx.trace) out.push(`条件を満たさなければ \`[skip:${step.name} <理由>]\` を出力して飛ばす。`);
  }
  if (step.whenExpr) out.push('', `**実行条件:** \`${prose(step.whenExpr)}\` が成り立つとき`);
  if (step.optional && ctx.trace) out.push('', `飛ばす場合は \`[skip:${step.name} <理由>]\` を出力する。`);
  if (step.reads.length) out.push('', `**先に読む:** ${step.reads.join('、')}`);
  if (step.rules.length) out.push('', `**特に守ること:** ${step.rules.join('、')}`);
  if (step.uses.length) out.push('', `**使うツール:** ${step.uses.map((c) => `\`${toolName(c, ctx)}\``).join('、')}`);
  if (step.gate !== undefined) {
    const g = renderGate(step, step.gate, ctx);
    if (g.length) out.push('', ...g);
  }
  const kind = renderKind(step, num, ctx, depth);
  if (kind.length) out.push('', ...kind);
  if (step.verify) out.push('', `**確認:** ${prose(step.verify)}。${step.onFail ? `満たさなければ手順「${step.onFail}」に戻る。` : ''}`);
  if (step.onError) {
    const e = step.onError;
    const then = e.then === 'continue' ? '失敗しても処理を続ける。' : e.then === 'goto' ? `失敗したら手順「${e.goto}」に進む。` : '失敗したら中止する。';
    out.push('', `**失敗したとき:** ${then}${e.say ? ` ユーザーには「${prose(e.say)}」と伝える。` : ''}`);
  }
  if (Object.keys(step.outputs).length) out.push('', '**この手順の出力:**', ...Object.entries(step.outputs).map(([k, v]) => `- ${k}: ${prose(v)}`));
  if (step.why) out.push('', `_理由: ${prose(step.why)}_`);
  out.push('');
  return out;
}

function renderStepList(steps: Step[], ctx: RenderContext, depth: number, prefix: string): string[] {
  return steps.flatMap((s, i) => renderStep(s, prefix ? `${prefix}.${i + 1}` : `${i + 1}`, ctx, depth));
}

/** トップレベルの手順列を文章化する。baseLevel は見出しの深さの起点（3 = ###） */
export function renderSteps(steps: Step[], ctx: RenderContext, baseLevel = 3): string[] {
  return renderStepList(steps, ctx, baseLevel - 3, '');
}

export function renderChecklist(steps: Step[]): string[] {
  return ['```text', '進捗:', ...steps.map((s, i) => `- [ ] ${i + 1}. ${s.name}`), '```'];
}
```

- [ ] **Step 4: 通ることを確認する**

Run: `npx vitest run tests/targets/render-steps.test.ts && npm run typecheck`
Expected: PASS（4件）、型エラーなし

- [ ] **Step 5: コミットする（ユーザーの承認を取ってから）**

```bash
git add src/targets/render-steps.ts tests/targets/render-steps.test.ts
git commit -m "feat(targets): 手順と制御構造を文章の手順に展開"
```

---

### Task 9: SKILL.md の組み立て

**Files:**
- Create: `src/targets/compile.ts`
- Test: `tests/targets/compile.test.ts`

- [ ] **Step 1: 失敗するテストを書く**

`tests/targets/compile.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { parse as parseYaml } from 'yaml';
import { parseSkill } from '../../src/parse/parse.js';
import { parseLocal, resolveSkill } from '../../src/resolve/resolve.js';
import { compileSkill } from '../../src/targets/compile.js';

const SKILL = `spec: skillrepro/v0.1
name: managing-tasks
title: タスク管理
invoke_as: タスク追加
description: |
  Manages tasks.
  Use when: the user adds or completes a task.
allowed-tools: [Read, Edit]
purpose: Keep tasks in one place.
requires:
  mcp:
    - slack.post: { description: Notify, default: mcp__slack__post }
knowledge:
  - { name: projects, description: Project list, content: "- alpha\\n- beta" }
  - { name: schema, description: File format, placement: file, content: "# Schema" }
rules:
  - { name: no-overwrite, description: Never overwrite existing lines. }
routes:
  - name: add
    description: |
      Add a task.
      Use when: adding.
    steps:
      - { name: write-task, description: Append the task., rules: [no-overwrite] }
  - name: complete
    description: Complete a task.
    steps:
      - { name: mark-done, description: Mark it done., uses: [slack.post] }
done_when: The task file is updated.
`;

const compile = (trace: boolean, target: 'claude-code' | 'claude-api' = 'claude-code') => {
  const { skill } = parseSkill(SKILL, 'skill.yaml');
  const resolved = resolveSkill(skill, parseLocal(''));
  return compileSkill(resolved, { target, trace });
};

describe('compileSkill', () => {
  it('uses invoke_as as the directory and writes valid frontmatter', () => {
    const out = compile(false);
    expect(out.dir).toBe('タスク追加');
    const md = out.files['SKILL.md']!;
    const fm = parseYaml(md.split('---')[1]!) as Record<string, unknown>;
    expect(fm.name).toBe('managing-tasks');
    expect(fm.description).toContain('Use when: the user adds');
    expect(fm['allowed-tools']).toEqual(['Read', 'Edit', 'mcp__slack__post']);
    expect(md).toContain('<!-- skill.yaml から生成。手編集禁止 -->');
  });

  it('renders rules, inline knowledge, routes and done_when in order', () => {
    const md = compile(false).files['SKILL.md']!;
    const order = ['# タスク管理', 'Keep tasks in one place.', '## 守ること', '- **no-overwrite**: Never overwrite existing lines.', '## 参照知識', '### projects', '## モードの判定', '| add | Add a task. Use when: adding. |', '## モード: add', '### 1. write-task', '## モード: complete', '## 完了条件'];
    const positions = order.map((s) => md.indexOf(s));
    expect(positions.every((p) => p >= 0)).toBe(true);
    expect([...positions].sort((a, b) => a - b)).toEqual(positions);
  });

  it('writes file-placed knowledge as a separate file linked one level deep', () => {
    const out = compile(false);
    expect(out.files['knowledge/schema.md']).toBe('# Schema\n');
    expect(out.files['SKILL.md']).toContain('- [schema](knowledge/schema.md): File format');
  });

  it('adds the marker instructions in trace mode', () => {
    const md = compile(true).files['SKILL.md']!;
    expect(md).toContain('## 観測用の目印');
    expect(md).toContain('判定したら `[route:<モード名>]` を出力する。');
    expect(compile(false).files['SKILL.md']).not.toContain('## 観測用の目印');
  });

  it('formats tool names for the API target', () => {
    const md = compile(false, 'claude-api').files['SKILL.md']!;
    expect(md).toContain('`slack:post`');
  });
});
```

- [ ] **Step 2: 失敗を確認する**

Run: `npx vitest run tests/targets/compile.test.ts`
Expected: FAIL（モジュールが見つからない）

- [ ] **Step 3: 実装する**

`src/targets/compile.ts`:

```ts
import { stringify } from 'yaml';
import type { Resolved } from '../resolve/resolve.js';
import { prose, renderChecklist, renderSteps, type RenderContext } from './render-steps.js';
import { formatTool, type Target } from './tools.js';

export interface CompileOptions {
  target: Target;
  trace: boolean;
}

export interface CompileOutput {
  /** 出力するディレクトリ名（invoke_as があればそれ、無ければ name） */
  dir: string;
  /** ディレクトリからの相対パス → 内容 */
  files: Record<string, string>;
}

const TRACE_GUIDE = [
  '## 観測用の目印',
  '',
  'このスキルは観測用に生成されています。手順を進めるときは、各手順に書かれた目印（`[step:...]` など）を、書かれたタイミングでそのまま1行で出力してください。目印は作業の結果とは別の行に出力し、省略しないでください。',
  '',
];

export function compileSkill(resolved: Resolved, options: CompileOptions): CompileOutput {
  const { skill, tools } = resolved;
  const ctx: RenderContext = { target: options.target, trace: options.trace, tools };
  const files: Record<string, string> = {};

  const allowed = [...new Set([...skill.allowedTools, ...Object.values(tools).map((t) => formatTool(t, options.target))])];
  const frontmatter = stringify({ name: skill.name, description: skill.description, 'allowed-tools': allowed }).trimEnd();

  const body: string[] = ['---', frontmatter, '---', '', '<!-- skill.yaml から生成。手編集禁止 -->', '', `# ${skill.title ?? skill.name}`, ''];
  if (skill.purpose) body.push(prose(skill.purpose), '');
  if (options.trace) body.push(...TRACE_GUIDE);

  if (skill.inputs.length) {
    body.push('## 入力', '', ...skill.inputs.map((i) => `- **${i.name}**: ${prose(i.description)}`), '');
  }
  if (skill.rules.length) {
    body.push('## 守ること', '', ...skill.rules.map((r) => `- **${r.name}**: ${prose(r.description)}`), '');
  }
  if (skill.knowledge.length) {
    body.push('## 参照知識', '');
    for (const k of skill.knowledge) {
      if (k.placement === 'file') {
        const path = `knowledge/${k.name}.md`;
        files[path] = `${k.content.trimEnd()}\n`;
        body.push(`- [${k.name}](${path}): ${prose(k.description)}`);
      }
    }
    for (const k of skill.knowledge) {
      if (k.placement === 'inline') body.push('', `### ${k.name}`, '', prose(k.description), '', k.content.trimEnd());
    }
    body.push('');
  }

  if (skill.routes) {
    body.push('## モードの判定', '', '依頼内容から、次のどのモードかを判定する。');
    if (options.trace) body.push('判定したら `[route:<モード名>]` を出力する。');
    body.push('', '| モード | 説明 |', '| --- | --- |');
    for (const r of skill.routes) body.push(`| ${r.name} | ${prose(r.description).replace(/\s*\n\s*/g, ' ')} |`);
    body.push('');
    for (const r of skill.routes) {
      body.push(`## モード: ${r.name}`, '', prose(r.description), '', ...renderChecklist(r.steps), '', ...renderSteps(r.steps, ctx));
    }
  } else if (skill.steps) {
    body.push('## 手順', '', ...renderChecklist(skill.steps), '', ...renderSteps(skill.steps, ctx));
  }

  if (skill.doneWhen) body.push('## 完了条件', '', prose(skill.doneWhen), '');

  files['SKILL.md'] = `${body.join('\n').replace(/\n{3,}/g, '\n\n').trimEnd()}\n`;
  return { dir: skill.invokeAs ?? skill.name, files };
}
```

- [ ] **Step 4: 通ることを確認する**

Run: `npx vitest run tests/targets/compile.test.ts && npm run typecheck`
Expected: PASS（5件）、型エラーなし

- [ ] **Step 5: コミットする（ユーザーの承認を取ってから）**

```bash
git add src/targets/compile.ts tests/targets/compile.test.ts
git commit -m "feat(targets): SKILL.md を組み立てる compileSkill"
```

---

### Task 10: 診断の表示と CLI

**Files:**
- Create: `src/diagnostics.ts`
- Create: `src/cli.ts`
- Test: `tests/cli.test.ts`

- [ ] **Step 1: 失敗するテストを書く**

`tests/cli.test.ts`:

```ts
import { mkdtempSync, readFileSync, writeFileSync, existsSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { describe, expect, it } from 'vitest';
import { main } from '../src/cli.js';
import { formatDiagnostic } from '../src/diagnostics.js';

const GOOD = 'spec: skillrepro/v0.1\nname: demo\ndescription: Demo. Use when testing.\nsteps:\n  - name: a\n    description: "Do A."\n';

function workspace(skillYaml: string) {
  const dir = mkdtempSync(join(tmpdir(), 'skillrepro-'));
  writeFileSync(join(dir, 'skill.yaml'), skillYaml);
  return dir;
}

function capture() {
  const lines: string[] = [];
  return { io: { out: (s: string) => lines.push(s), err: (s: string) => lines.push(s) }, lines };
}

describe('formatDiagnostic', () => {
  it('prints where, what and how to fix', () => {
    expect(
      formatDiagnostic({ file: 'skill.yaml', line: 4, severity: 'error', rule: 'missing-name', message: 'name がありません。', fix: 'name を足す。' }),
    ).toBe('skill.yaml:4  error  missing-name\n  name がありません。\n  直し方: name を足す。');
  });
});

describe('skillrepro compile', () => {
  it('writes SKILL.md and exits 0', async () => {
    const dir = workspace(GOOD);
    const out = join(dir, 'out');
    const { io } = capture();
    const code = await main(['compile', join(dir, 'skill.yaml'), '--target', 'claude-code', '--out', out], io);
    expect(code).toBe(0);
    expect(readFileSync(join(out, 'demo', 'SKILL.md'), 'utf8')).toContain('### 1. a');
  });

  it('prints diagnostics, writes nothing and exits 1 on errors', async () => {
    const dir = workspace('spec: skillrepro/v0.1\nname: demo\ndescription: d\nsteps:\n  - run: ls\n');
    const out = join(dir, 'out');
    const { io, lines } = capture();
    const code = await main(['compile', join(dir, 'skill.yaml'), '--out', out], io);
    expect(code).toBe(1);
    expect(lines.join('\n')).toContain('missing-name');
    expect(existsSync(out)).toBe(false);
  });

  it('reads skill.local.yaml next to skill.yaml', async () => {
    const dir = workspace(GOOD.replace('Do A.', 'Read {{paths.notes}}.') + 'requires:\n  paths:\n    - notes: where notes are\n');
    writeFileSync(join(dir, 'skill.local.yaml'), 'paths:\n  notes: work/notes\n');
    const out = join(dir, 'out');
    const { io } = capture();
    expect(await main(['compile', join(dir, 'skill.yaml'), '--out', out], io)).toBe(0);
    expect(readFileSync(join(out, 'demo', 'SKILL.md'), 'utf8')).toContain('Read work/notes.');
  });

  it('rejects unsupported targets', async () => {
    const dir = workspace(GOOD);
    const { io, lines } = capture();
    expect(await main(['compile', join(dir, 'skill.yaml'), '--target', 'codex'], io)).toBe(1);
    expect(lines.join('\n')).toContain('対応していない出力先です: codex');
  });
});
```

- [ ] **Step 2: 失敗を確認する**

Run: `npx vitest run tests/cli.test.ts`
Expected: FAIL（モジュールが見つからない）

- [ ] **Step 3: diagnostics.ts を実装する**

`src/diagnostics.ts`:

```ts
import type { Diagnostic } from './model.js';

export function formatDiagnostic(d: Diagnostic): string {
  const lines = [`${d.file}:${d.line}  ${d.severity}  ${d.rule}`, `  ${d.message}`];
  if (d.fix) lines.push(`  直し方: ${d.fix}`);
  return lines.join('\n');
}

export const hasErrors = (ds: Diagnostic[]) => ds.some((d) => d.severity === 'error');
```

- [ ] **Step 4: cli.ts を実装する**

`src/cli.ts`:

```ts
#!/usr/bin/env node
import { existsSync, mkdirSync, readFileSync, writeFileSync } from 'node:fs';
import { dirname, join, resolve } from 'node:path';
import { fileURLToPath } from 'node:url';
import { parseArgs } from 'node:util';
import { formatDiagnostic, hasErrors } from './diagnostics.js';
import { parseSkill } from './parse/parse.js';
import { parseLocal, resolveSkill } from './resolve/resolve.js';
import { compileSkill } from './targets/compile.js';
import { TARGETS, parseTarget } from './targets/tools.js';

export interface Io {
  out: (s: string) => void;
  err: (s: string) => void;
}

const USAGE = `使い方:
  skillrepro compile <skill.yaml> [--target claude-code|claude-api] [--trace] [--out <dir>]`;

async function compileCommand(args: string[], io: Io): Promise<number> {
  const { values, positionals } = parseArgs({
    args,
    allowPositionals: true,
    options: { target: { type: 'string', default: 'claude-code' }, trace: { type: 'boolean', default: false }, out: { type: 'string', default: 'dist' } },
  });
  const target = parseTarget(values.target);
  if (!target) {
    io.err(`対応していない出力先です: ${values.target}（使えるもの: ${TARGETS.join(', ')}）`);
    return 1;
  }
  const file = resolve(positionals[0] ?? 'skill.yaml');
  if (!existsSync(file)) {
    io.err(`skill.yaml が見つかりません: ${file}`);
    return 1;
  }
  const { skill, diagnostics } = parseSkill(readFileSync(file, 'utf8'), file);
  const localFile = join(dirname(file), 'skill.local.yaml');
  const local = parseLocal(existsSync(localFile) ? readFileSync(localFile, 'utf8') : '');
  const resolved = hasErrors(diagnostics) ? undefined : resolveSkill(skill, local);
  const all = [...diagnostics, ...(resolved?.diagnostics ?? [])];
  for (const d of all) io.err(formatDiagnostic(d));
  if (!resolved || hasErrors(all)) return 1;
  for (const n of resolved.notices) io.err(`注意: ${n}`);

  const output = compileSkill(resolved, { target, trace: values.trace });
  const outDir = join(resolve(values.out), output.dir);
  for (const [path, content] of Object.entries(output.files)) {
    const dest = join(outDir, path);
    mkdirSync(dirname(dest), { recursive: true });
    writeFileSync(dest, content);
  }
  io.out(`生成しました: ${join(outDir, 'SKILL.md')}${values.trace ? '（観測用）' : ''}`);
  return 0;
}

export async function main(argv: string[], io: Io): Promise<number> {
  const [command, ...rest] = argv;
  if (command === 'compile') return compileCommand(rest, io);
  io.err(USAGE);
  return command === undefined || command === '--help' ? 0 : 1;
}

if (process.argv[1] && resolve(process.argv[1]) === fileURLToPath(import.meta.url)) {
  const code = await main(process.argv.slice(2), { out: (s) => console.log(s), err: (s) => console.error(s) });
  process.exit(code);
}
```

- [ ] **Step 5: 通ることを確認する**

Run: `npx vitest run tests/cli.test.ts && npm run typecheck`
Expected: PASS（5件）、型エラーなし

- [ ] **Step 6: コミットする（ユーザーの承認を取ってから）**

```bash
git add src/diagnostics.ts src/cli.ts tests/cli.test.ts
git commit -m "feat(cli): skillrepro compile コマンド"
```

---

### Task 11: 見本スキルとスナップショット

**Files:**
- Create: `examples/weekly-digest/skill.yaml`
- Create: `examples/weekly-digest/skill.local.example.yaml`
- Create: `examples/managing-tasks/skill.yaml`
- Test: `tests/examples.test.ts`

- [ ] **Step 1: 直列の見本を作る**

`examples/weekly-digest/skill.yaml`:

```yaml
spec: skillrepro/v0.1
name: weekly-digest
title: Weekly digest
description: |
  Summarizes this week's meeting notes and posts decisions and action items to Slack.
  Use when: the user asks for a weekly digest of meetings.
allowed-tools: [Read, Glob, Bash]
purpose: |
  Keeps decisions from getting buried in a busy week of meetings.
requires:
  tools:
    - python3: "Runs the bundled script"
  mcp:
    - slack.post:
        description: "Posts the digest"
        default: mcp__slack__slack_post_message
  env:
    - SLACK_CHANNEL: "Channel ID to post to"
  paths:
    - notes_dir: "Folder that holds meeting notes"
rules:
  - name: cite-sources
    description: Every decision cites the meeting note it came from.
steps:
  - name: collect
    description: Collect this week's meeting notes, skipping empty ones.
    run: python3 scripts/list_notes.py --dir {{paths.notes_dir}} --days 7
    freedom: low
    why: Empty notes from failed transcriptions must be excluded deterministically.
    outputs:
      notes: the list of note files
  - name: summarize
    description: Extract decisions and action items from {{steps.collect.outputs.notes}}. Write "unassigned" when no owner is named.
    rules: [cite-sources]
    verify: every decision has a source note
    on_fail: summarize
  - name: post
    description: Post the digest to {{env.SLACK_CHANNEL}}.
    call:
      capability: slack.post
      args: { channel: "$SLACK_CHANNEL" }
    uses: [slack.post]
    gate: confirm
done_when: |
  The post URL is returned together with the list of notes that were used.
```

`examples/weekly-digest/skill.local.example.yaml`:

```yaml
# skill.local.yaml にコピーして、自分の環境の値に書き換える（skill.local.yaml は git に入れない）
paths:
  notes_dir: notes/meetings
```

- [ ] **Step 2: 分岐・対話・知識の見本を作る**

`examples/managing-tasks/skill.yaml`:

```yaml
spec: skillrepro/v0.1
name: managing-tasks
title: Managing tasks
description: |
  Adds, completes and lists tasks kept in a single Markdown file.
  Use when: the user says "add this as a task", "done", or "what's left".
allowed-tools: [Read, Edit]
knowledge:
  - name: task-format
    description: One task per line, as a Markdown checkbox followed by the project in brackets.
    content: |
      - [ ] Draft the proposal [alpha]
      - [x] Book the venue [beta]
rules:
  - name: no-overwrite
    description: Never rewrite an existing task line except to tick its checkbox.
routes:
  - name: add
    description: |
      Adds a new task.
      Use when: the user wants something tracked as a task.
    steps:
      - name: match-project
        description: Decide which project the task belongs to from the request.
        reads: [task-format]
      - name: confirm-project
        description: Ask only when no project matches.
        when: no project matches the request
        ask:
          question: Which project is this task for?
          choices: [alpha, beta]
      - name: append-task
        description: Append the task to tasks.md.
        rules: [no-overwrite]
        gate: none
        why: Appending is safe and reversible, so no confirmation is needed.
  - name: complete
    description: |
      Marks a task as done.
      Use when: the user says a task is finished.
    steps:
      - name: tick-task
        description: Tick the checkbox of the matching task in tasks.md.
        rules: [no-overwrite]
  - name: list
    description: |
      Lists open tasks.
      Use when: the user asks what is left.
    steps:
      - name: show-open
        description: Show unticked tasks grouped by project.
done_when: tasks.md reflects the request, and the change is shown to the user.
```

- [ ] **Step 3: 失敗するテストを書く**

`tests/examples.test.ts`:

```ts
import { readFileSync } from 'node:fs';
import { describe, expect, it } from 'vitest';
import { parseSkill } from '../src/parse/parse.js';
import { parseLocal, resolveSkill } from '../src/resolve/resolve.js';
import { compileSkill } from '../src/targets/compile.js';

const EXAMPLES = [
  { name: 'weekly-digest', local: 'examples/weekly-digest/skill.local.example.yaml' },
  { name: 'managing-tasks', local: undefined },
];

describe.each(EXAMPLES)('examples/$name', ({ name, local }) => {
  const file = `examples/${name}/skill.yaml`;
  const { skill, diagnostics } = parseSkill(readFileSync(file, 'utf8'), file);
  const resolved = resolveSkill(skill, parseLocal(local ? readFileSync(local, 'utf8') : ''));

  it('has no diagnostics', () => {
    expect(diagnostics).toEqual([]);
    expect(resolved.diagnostics).toEqual([]);
  });

  it.each([false, true])('compiles (trace=%s) to a stable SKILL.md', (trace) => {
    const out = compileSkill(resolved, { target: 'claude-code', trace });
    expect(out.files['SKILL.md']).toMatchSnapshot();
  });
});
```

- [ ] **Step 4: テストを実行してスナップショットを作る**

Run: `npx vitest run tests/examples.test.ts`
Expected: PASS。`tests/__snapshots__/examples.test.ts.snap` が作られる

- [ ] **Step 5: スナップショットを目で確認する**

`tests/__snapshots__/examples.test.ts.snap` を開き、次を確認する。問題があれば該当の Task に戻って直し、`npx vitest run -u` で作り直す。

- weekly-digest: `collect` のコマンドの `{{paths.notes_dir}}` が `notes/meetings` に置き換わっている
- weekly-digest: `summarize` の説明が「手順「collect」の出力 notes」になっている
- weekly-digest: `post` に「実行前に、ユーザーの承認を取る」がある。`$SLACK_CHANNEL` のまま値が入っていない
- managing-tasks: 「モードの判定」の表に add / complete / list の3行がある
- trace=true のほうにだけ `[step:` `[route:` `[ask:` の目印の指示がある

- [ ] **Step 6: 全体を確認する**

Run: `npm test && npm run typecheck && npm run build && node dist/cli.js compile examples/managing-tasks/skill.yaml --trace --out /tmp/skillrepro-check`
Expected: 全テスト PASS、型エラーなし、`生成しました: /tmp/skillrepro-check/managing-tasks/SKILL.md（観測用）`

- [ ] **Step 7: コミットする（ユーザーの承認を取ってから）**

```bash
git add examples tests/examples.test.ts tests/__snapshots__
git commit -m "test: 見本スキル2本と SKILL.md のスナップショット"
```
