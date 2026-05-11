# DR-0001: abstract module を `extends` で実装する

- Status: Active
- Date: 2026-05-11

## Context

`tasks/vcs/iface.pkl` を「VCS 抽象インターフェース」として宣言し、`tasks/vcs/jj.pkl` / `tasks/vcs/git.pkl` がその実装を提供する設計。Pkl にはモジュール継承の構文として `amends` と `extends` の 2 つがあり、当初は「abstract module + amends」で実装する想定だった (README にもそう書いていた)。

Pkl の `amends` は **既存モジュールを継承して値を上書き** する用途。一方 `extends` は **abstract モジュールの未実装プロパティを実装する** 用途で、Java の `extends` に近いセマンティクス。実際に `amends` で `abstract module` を継承しようとすると Pkl 評価時に `Cannot instantiate abstract class` で停止する。

## Decision

`abstract module com.github.kawaz.pkfTasksVcsIface` を宣言する側 (`tasks/vcs/iface.pkl`) と、実装する側 (`tasks/vcs/jj.pkl` / `git.pkl` / `auto.pkl`) の関係は **`extends "iface.pkl"`** で表現する。`amends` ではない。

```pkl
// iface.pkl
abstract module com.github.kawaz.pkfTasks.vcs.iface
abstract commit: Taskfile.Task
abstract push: Taskfile.Task
abstract fetch: Taskfile.Task
abstract ensureClean: Taskfile.Task

// jj.pkl
module com.github.kawaz.pkfTasks.vcs.jj
extends "iface.pkl"

commit = new Taskfile.Task { name = "vcs:commit"; cmd = "..." }
push = new Taskfile.Task { name = "vcs:push"; cmd = "..." }
// ... (abstract プロパティを全部埋める)
```

## Rationale

### 不採用案

**1. `amends "iface.pkl"` で実装する**

Pkl 公式の `amends` は「既存モジュールを継承して特定プロパティだけ override」用途。**不採用**: abstract module を amends すると `Cannot instantiate abstract class` で評価失敗する (Pkl 0.31 で確認)。abstract モジュールは具象化が必要な「Java の interface」相当で、`amends` の「具象を継承」モデルと相容れない。

**2. abstract 宣言を外して plain な module + amends で実装する**

iface.pkl から `abstract` を外し、各プロパティに `null` 等の default 値を置く案。**不採用**: `null` がそのまま render される事故 (実装側で上書きし忘れる → Task.name が `null` になって pkfire が起動時に error) のリスクを引き受けることになる。abstract module + extends だと「実装漏れは Pkl 評価時に error」で型システムが守ってくれる。

### 設計上のポイント

#### `extends` の選択は不可避

これは「設計選好」というより Pkl 仕様による必然。abstract module を実装する手段は `extends` のみ。

#### Module-level vs class-level

Pkl は class でも abstract が使えるが、`Task` 自体は pkfire の class なので、利用側で別 class を定義する必要はない。**プロパティ宣言を module レベルで abstract にする** だけで十分。

## Consequences

- `iface.pkl` を単体 `pkl eval` すると abstract プロパティが null として render され、Pkl 内部の PCF render 経路で `Cannot convert VM value with unexpected type: null` で失敗する (Pkl 0.31 の既知挙動)。実害なし: iface.pkl を直接 eval する運用は無い。実装側 (`jj.pkl` / `git.pkl` / `auto.pkl`) は eval 成功する
- README / DESIGN で **「amends」と書かないこと**。利用者が真似ようとして詰まる
- 将来 abstract module ではなく concrete + override パターンに切り替える時は本 DR の Superseded-by を立てる

## 関連

- 実装: `tasks/vcs/iface.pkl` / `jj.pkl` / `git.pkl` / `auto.pkl`
- 反映ドキュメント: `README.md` / `README-ja.md` の Design セクション
