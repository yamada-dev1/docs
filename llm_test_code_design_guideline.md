# LLM向けテストコード設計ガイドライン

## 1. 目的

本ガイドラインは、LLMに単体テスト、結合テスト、UIテスト、E2Eテスト等を生成させる際に、単なるコードカバレッジではなく、**仕様・要求された役割・振る舞いに対するテストの妥当性と網羅性**を確保するための設計方針を定める。

基本方針は次のとおりとする。

> 仕様から直接テストコードを生成しない。
> まず「何を保証すべきか」をテスト設計上の中間成果物として明確化し、その後にテストレベルとテストケースを設計してからコードを生成する。

---

## 2. 基本原則

### 2.1 コードカバレッジを「十分性」の尺度にしない

Line Coverage / Branch Coverage は、未実行コードや未実行分岐を見つけるために利用する。

ただし、100%になったことをもってテスト十分とは判断しない。

テストの十分性は、少なくとも以下を組み合わせて判断する。

- Requirement / Requirement Obligation Coverage
- Behavior Coverage
- Decision Coverage
- Boundary Coverage
- State / Transition Coverage
- Error / Negative Coverage
- Contract Coverage
- Invariant / Property Coverage
- Code Coverage
- Mutation Score

特に重要なのは、**仕様上必要な振る舞いをテストできているか**と、**テストが実際に欠陥を検出できるか**である。

### 2.2 テスト観点とテストコードを1対1にしない

以下の関係を基本とする。

```text
テスト観点
  ↓
Test Obligation
  ↓
Test Case
  ↓
Test Method / Test Scenario
```

1つのテストが複数のTest Obligationを満たしてもよい。

例えば、以下を1つのUnit Testで同時に確認できる場合は、無理に3本へ分割しない。

- 在庫超過で引当が失敗する
- 失敗後も注文状態が維持される
- 失敗後も在庫が変化しない

一方、テストレベル・責務・失敗原因が異なる場合は分離する。

### 2.3 コード上の行・if文をテスト単位にしない

「1分岐 = 1テスト」「1行 = 1テスト」のような生成は行わない。

ソースコードはテスト観点を発見するための情報の一つであり、テストコードの粒度は、**仕様上の責務・Test Obligation・失敗原因・テストレベル**から決定する。

### 2.4 生成と評価を分離する

LLMによるテスト生成だけで品質を完結させない。

```text
仕様・実装
  ↓
テスト設計LLM
  ↓
Test Obligation / Test Case
  ↓
コード生成LLM
  ↓
テストコード
  ↓
機械的検証
  ├─ Coverage
  ├─ Mutation
  ├─ Contract
  └─ Architecture
  ↓
不足・不整合を検出
  ↓
LLMで追加・修正
```

---

# 3. テストコード作成前の標準工程

テストコードを生成する前に、次の6段階を経る。

```text
仕様
 ↓
① Behavior Extract
 ↓
② Test Perspective
 ↓
③ Test Model
 ↓
④ Test Obligation
 ↓
⑤ Test Level Assignment
 ↓
⑥ Test Case Design
 ↓
テストコード生成
```

---

## 3.1 ① Behavior Extract — 仕様から振る舞いを抽出する

まず仕様を読んで、「対象機能が何をするのか」を整理する。

入力例:

```text
在庫引当を実行する。
引当数量が在庫数量以下なら成功する。
在庫を超える場合はエラーとし、在庫を変更しない。
引当済み注文は再度引当できない。
```

抽出結果の例:

```yaml
feature: 在庫引当
behaviors:
  - 在庫以内なら引当成功
  - 在庫超過なら引当失敗
  - 成功時は在庫を減算
  - 失敗時は在庫を変更しない
  - 引当済み注文は再引当できない
```

この段階ではまだJUnitやPlaywrightのコードを作らない。

---

## 3.2 ② Test Perspective — 何を見るべきかを洗い出す

Behaviorに対して、どのような観点を検討するかを列挙する。

標準観点として次を使用する。

| 観点 | 主な確認対象 |
|---|---|
| 正常系 | 正常な入力・状態での期待動作 |
| 異常系 | 不正入力・業務エラー・例外 |
| 境界値 | 最小、最大、境界の前後 |
| 条件組合せ | 複数条件による結果の違い |
| 状態 | 各状態で許される・許されない操作 |
| 状態遷移 | 状態変化、禁止遷移 |
| 入出力 | 入力と結果の対応 |
| 副作用 | DB、イベント、外部I/F等への変更 |
| 不変条件 | 常に成立すべき条件 |
| 契約 | API / メッセージ / I/Fの契約 |
| 権限 | ロール・権限による制御 |
| UI状態 | 表示、入力、disabled、loading、error等 |
| 業務シナリオ | 複数操作を跨ぐ業務フロー |
| 回復・再実行 | Retry、再送、二重実行等 |
| 並行性 | 同時更新・競合が意味を持つ場合 |

全機能で全観点を機械的にテストするのではなく、**適用対象かどうかを判断した証跡を残す**ことが重要である。

---

# 4. Test Model

Test Perspectiveを具体的なモデルへ落とし込む。

## 4.1 Gherkin / Scenario

Gherkinは主に、**業務ルール・振る舞い・受入条件を具体例として表現する**ために利用する。

```gherkin
Feature: 在庫引当

  Rule: 在庫を超える引当はできない

    Scenario: 在庫以内で引き当てる
      Given 注文が未引当である
      And 在庫が10個ある
      When 5個を引き当てる
      Then 引当は成功する
      And 在庫は5個になる
```

Gherkinはテストコードそのものではなく、**Behaviorの基準**として扱う。

単体テストをすべてGherkin化する必要はない。

---

## 4.2 Decision Table

Decision Tableは、**複数条件の組合せと、それによる結果**を整理するために使う。

例:

| 注文状態 | 数量 | 在庫 | 結果 |
|---|---:|---|---|
| 未引当 | 正 | 十分 | 成功 |
| 未引当 | 正 | 不足 | 失敗 |
| 未引当 | 0 | 十分 | 入力エラー |
| 未引当 | 負 | 十分 | 入力エラー |
| 引当済 | 正 | 十分 | 再引当エラー |

注意事項:

- 全組合せを無条件にテストしない。
- 不可能な組合せを除外する。
- 同値なルールをまとめる。
- リスクの高い組合せを優先する。
- 必要に応じてPairwise等で組合せ数を抑える。

Decision Tableは通常、単独の実行ライブラリに依存させず、JUnit Parameterized Test、Gherkin Scenario Outline等の入力モデルとして扱う。

---

## 4.3 State Model

State Modelは、**状態と状態遷移**を明示するために利用する。

例:

```text
未引当
  ├─ 引当成功 → 引当済
  └─ 引当失敗 → 未引当

引当済
  └─ 再引当 → エラー
```

最低限、次を定義する。

- 状態
- イベント / 操作
- 遷移条件
- 遷移後状態
- 遷移時の副作用
- 禁止遷移

正常遷移だけでなく、**禁止遷移を明示してテスト対象とする**。

State Modelは必ずしも専用ライブラリへ依存する必要はない。YAML / JSON等のモデルとして保持し、JUnit、Gherkin、Playwright等へ展開できる構成を基本とする。

---

## 4.4 Contract

Contractは、**システム境界で守るべき契約**を表現する。

REST APIであれば、例えば次を含む。

- HTTP method
- URL
- request fields
- response fields
- required / optional
- 型
- status code
- error response
- 必要なheaders

OpenAPIが存在する場合は有力なContractソースになる。

PactはConsumer Driven Contract Testingの代表的な実装候補であり、Consumer側の期待するinteractionからcontractを生成し、Provider側で検証する。APIやサービス間連携で特に有効である。 citeturn650908search0turn650908search5

Contract Testは内部ロジックの正しさを検証するUnit Testとは役割を分ける。

---

## 4.5 Property / Invariant

Property / Invariantは、具体例ではなく、**前提条件のもとで常に成立すべき性質**を表す。

例:

```text
引当後在庫 >= 0
```

```text
引当失敗時、在庫数量は変更されない
```

Javaではjqwikが候補となる。jqwikはJUnit 5 Platform上でProperty-Based Testingを実行でき、生成データによる多数の試行と失敗ケースの縮小をサポートする。 citeturn399554search1

Property TestはExample-based Unit Testの代替ではなく、**具体例では表現しにくい一般則を補完するもの**とする。

---

# 5. Test Obligation

Test Obligationとは、

> **その機能について、テストによって保証しなければならない具体的な内容**

である。

例:

| ID | Test Obligation |
|---|---|
| TO-INV-001 | 在庫以内の引当が成功する |
| TO-INV-002 | 在庫超過の引当が拒否される |
| TO-INV-003 | 数量0が拒否される |
| TO-INV-004 | 負数が拒否される |
| TO-INV-005 | 引当成功時に在庫が減少する |
| TO-INV-006 | 引当失敗時に在庫が変更されない |
| TO-INV-007 | 引当成功時に注文状態が引当済になる |
| TO-INV-008 | 引当済注文を再引当できない |
| TO-INV-009 | 引当後在庫が負にならない |
| TO-INV-010 | APIが定義されたエラー契約を満たす |

各Obligationには可能な限り根拠を紐付ける。

```yaml
id: TO-INV-006
statement: 引当失敗時に在庫が変更されない
source:
  - requirement: REQ-INV-012
  - decision_table: DT-INV-001/R2
  - gherkin: allocation.feature#scenario-2
```

この紐付けを使って、要求→テスト→コードのトレーサビリティを構築する。

---

# 6. Test Level Assignment

各Test Obligationについて、どのテストレベルで保証するかを決定する。

| Test Obligation | Unit | Integration | UI | E2E |
|---|---:|---:|---:|---:|
| 引当計算 | ◎ |  |  |  |
| 在庫超過拒否 | ◎ |  |  | ○ |
| DB更新 |  | ◎ |  |  |
| API契約 |  | ◎ |  | ○ |
| ボタンdisabled |  |  | ◎ |  |
| 引当業務フロー |  |  |  | ◎ |
| 在庫非負 | ◎ |  |  |  |
| 状態遷移 | ◎ | ○ |  | ○ |

原則として、**同じ保証を全レベルで重複して実装しない**。

E2Eは業務フローの成立確認に集中させ、細かな計算・境界・不変条件は可能な限りUnit / Propertyへ寄せる。

---

# 7. Test Case Design

Test Obligationから、具体的なテストケースを設計する。

```text
TC-INV-001
対象: AllocationService
Given: 注文=未引当、在庫=10
When: 引当数量=5
Then: 結果=成功、在庫=5、注文状態=引当済
```

```text
TC-INV-002
対象: AllocationService
Given: 注文=未引当、在庫=10
When: 引当数量=11
Then: 結果=失敗、在庫=10、注文状態=未引当
```

この段階では、まだJavaやTypeScriptのテストコードを書かない。

---

# 8. テストコードの粒度

## 8.1 基本単位

テストコードは「観点」ではなく、**テストケースと責務**を中心に設計する。

例えば、Decision Tableの5行をそのまま5個の独立したメソッドにする必要はない。

同じ対象・同じ検証目的・同じfixtureであれば、Parameterized Testにまとめる。

```text
Decision Table
      ↓
複数Test Case
      ↓
1つのParameterized Test
```

## 8.2 まとめる条件

次の条件を満たす場合、複数観点を1つのTest Method / Test Scenarioにまとめてよい。

- 同じテストレベル
- 同じ対象コンポーネント
- 同じ業務シナリオ
- 同じ失敗原因
- 同じfixtureを自然に共有できる
- 複数assertionが1つの目的を補強している

## 8.3 分ける条件

次の場合は分ける。

- Unit / Integration / E2E等、テストレベルが異なる
- 失敗原因が異なる
- テスト対象の責務が異なる
- Property TestとExample-based Testで目的が異なる
- Contractと内部ロジックで目的が異なる
- 失敗時の原因特定が難しくなる
- setup / fixtureが大きく異なる

---

# 9. テストレベル別の役割と主な入力

| テスト | 主な目的 | 主な設計入力 | 主な実行基盤 |
|---|---|---|---|
| Unit | クラス・メソッドの責務、業務ロジック | Test Obligation、Decision Table、Property、ソース | JUnit 5 + AssertJ |
| Property | 一般則・不変条件 | Property / Invariant | jqwik |
| Integration | コンポーネント間連携 | Test Obligation、Contract、Gherkin、Integration仕様 | JUnit + Testcontainers等 |
| API / Contract | 境界契約 | OpenAPI、Contract、Gherkin | Pact / REST Assured等 |
| UI / Component | UIコンポーネントの振る舞い | UI仕様、State Model、Gherkin | Angular TestBed / Vitest / Playwright等 |
| E2E | 業務シナリオ全体 | Gherkin、Business Flow、State Model、Contract | Playwright |
| Architecture | 構造・依存関係 | Architecture Rule | ArchUnit |

### 注意

表中のツールは「必ず全部導入する」という意味ではない。**テスト目的に対する候補**であり、導入は対象プロジェクトの構成・コスト・既存ツールを考慮する。

---

# 10. Java系での基本ツール候補

## 採用優先度が高い基盤

### JUnit 5

JavaのUnit / Integration Testの基本ランナーとして使用する。

### AssertJ

assertionを可読性高く記述するための補助として使用する。

### Testcontainers

実DB・外部サービス等をコンテナで立ち上げてIntegration Testを行う場合の候補。JUnit 5との統合が提供されている。 citeturn650908search13turn650908search15

### jqwik

Property-based Testが有効なロジックに限定して使用する。すべてのUnit TestをProperty Testに置き換えない。 citeturn399554search1

### PIT

Mutation Testingによって、テストが欠陥を検出できるかを評価する。PITではMutationがKilled / Survived等に分類され、SurvivedはそのMutationをテストが検出できなかったことを意味する。 citeturn399554search6

### ArchUnit

Javaのpackage / class / layer / slice等の依存関係や循環を、Unit Testに近い形で検証する。 citeturn650908search12

---

# 11. Gherkin / Cucumberの位置付け

Cucumber / Gherkinは、主にBusiness Rule / Behavior / Acceptance Scenarioの表現として利用する。

GherkinのRuleは1つのBusiness Ruleを表し、Scenario / Exampleはその具体例を表す。ScenarioはGiven / When / Thenで初期条件・イベント・期待結果を表現する。 citeturn604184search11

したがって、Gherkinを次のように位置付ける。

```text
Requirement
    ↓
Business Rule / Behavior
    ↓
Gherkin
    ↓
Test Obligation
    ↓
Unit / Integration / E2Eへ展開
```

**Unit Testの全件をGherkin化することは推奨しない。**

---

# 12. Contract Testingの位置付け

Contract Testは、内部ロジックではなく「利用者と提供者の境界」を保証する。

例:

```text
Angular
   ↓ REST
Java API
```

に対して、

- request / response
- field / type
- required / optional
- status
- interaction

を保証する。

PactはConsumer側の期待するinteractionからContractを生成し、Provider側でそのContractを検証するConsumer Driven Contract Testingの代表的な選択肢である。 citeturn650908search0turn650908search1

OpenAPIがある場合は、OpenAPI自体をContractの主要な情報源として扱い、Contract Testと組み合わせる。

---

# 13. UI / E2E: Playwright

PlaywrightはUI / E2Eの標準候補とする。

主な役割:

- ブラウザ操作
- 入力 / 選択 / click
- navigation
- assertion
- screenshot
- trace
- network / consoleの調査
- Chromium / Firefox / WebKit等での実行

Playwright Testはデフォルトでheadless実行できる。 citeturn604184search4turn604184search9

## 開発コンテナでの基本インストール

今回の標準環境では、Chromiumのheadless shellを使用する。

```bash
npm install -D @playwright/test
npx playwright install --with-deps chromium --only-shell
```

Playwrightは通常、Playwright管理のChromiumを利用するため、Google Chrome等を別途インストールする必要はない。`--only-shell`を付けると、full ChromiumではなくChromium headless shellのみを導入できる。 citeturn604184search0turn604184search2

通常実行:

```bash
npx playwright test
```

を標準とする。

### headed / Codegenについて

Codegenは必須工程ではない。LLMへHTML、Angular template、component code等を渡してテストを生成することを基本とし、Codegenは実際のrendered DOMやlocatorの確認、UI操作の初期記録が必要な場合に使う。

Playwright Codegenは、実際のページを操作しながらPlaywrightコードを生成し、role / text / test id等を優先したlocatorを提案する。 citeturn604184search1

`--only-shell`はheadless専用の標準環境とするため、headed / Codegenが必要な開発時だけ通常Chromiumを追加導入するか、別のUIデバッグ環境を利用する。

---

# 14. LLMに渡す入力

テストコード生成時には、可能な限り次の情報を分離して与える。

## 14.1 Specification Input

```text
Requirement
Business Rule
Gherkin
Decision Table
State Model
Contract
Property / Invariant
```

## 14.2 Implementation Input

```text
Java source
Angular component
HTML template
API client
OpenAPI
関連DTO / Model
既存テスト
```

## 14.3 Existing Test Input

```text
既存Test Code
既存Test Obligation
既存のTraceability
過去の失敗
```

LLMは実装だけでなく、**仕様モデルと既存テストの両方**を入力として使う。

---

# 15. LLMの役割を分割する

LLMへの依頼を1回で終わらせない。

## Phase A: Test Perspective生成

```text
仕様
 +
実装
 ↓
LLM
 ↓
Test Perspective
```

## Phase B: Test Model生成 / 更新

```text
Test Perspective
 ↓
Decision Table
State Model
Contract
Property
Gherkin
```

## Phase C: Test Obligation生成

```text
Test Model
 ↓
Test Obligation
```

## Phase D: Test Level / Suite Design

```text
Test Obligation
 ↓
Unit / Integration / UI / E2Eへの割当
 ↓
Test Suite Design
```

## Phase E: Test Case Design

```text
Test Suite Design
 ↓
Test Case
```

## Phase F: Code Generation

ここで初めて、

```text
JUnit
jqwik
REST Assured / Pact
Playwright
```

等の実コードを生成する。

---

# 16. テストコード生成の品質評価

生成後は、Test Obligationに対して実装されたTest Caseを照合する。

例:

| Obligation | Test Case | Status |
|---|---|---|
| TO-001 | TC-001 | ✓ |
| TO-002 | TC-002 | ✓ |
| TO-003 | TC-003 | ✓ |
| TO-004 | TC-004 | ✓ |
| TO-005 | TC-001 | ✓ |
| TO-006 | TC-002 | ✓ |
| TO-007 | TC-001 | ✓ |
| TO-008 | - | ✗ |
| TO-009 | Property-001 | ✓ |
| TO-010 | Contract-001 | ✓ |

ここで未対応のObligationがあれば、コードカバレッジとは独立して「仕様上のテスト不足」と判定する。

---

# 17. Mutation Testingをフィードバックループに使う

Mutation Testingは単なる品質レポートとしてではなく、**不足観点を発見するためのフィードバック**として使う。

```text
Test Code
   ↓
PIT
   ↓
Mutation Survived
   ↓
LLMにMutation内容を提示
   ↓
不足しているTest Obligationを分析
   ↓
Test Case追加
   ↓
再実行
```

例えば、

```text
quantity > stock
```

を

```text
quantity >= stock
```

へMutationしてもテストが成功してしまった場合、在庫数量と引当数量が同値となる境界条件の不足を疑う。

Mutation Scoreを単独の合格基準にするのではなく、**不足観点発見の補助情報**として利用する。

---

# 18. Coverageの位置付け

Coverageは次の目的に限定する。

```text
「実行されていないコード・分岐を見つける」
```

Coverageが低い場合はテスト追加を検討する。

Coverageが高くても、以下が不足している可能性がある。

- 妥当なExpected Resultになっていない
- 境界条件がない
- 禁止遷移がない
- Contractが検証されていない
- Propertyが検証されていない
- 実装と同じロジックでExpectedを計算している

したがって、CoverageはTest Obligation CoverageやMutationと併用する。

---

# 19. Expected Result / Test Oracleに関する原則

LLMが実装コードを見てExpected Resultを同じロジックから作ることを避ける。

悪い例:

```text
実装:
  price * quantity

テスト:
  expected = price * quantity
```

これは実装と同じバグを再現する可能性がある。

可能な限り、次を利用する。

- 仕様に記載された期待値
- 独立した計算
- Invariant
- Contract
- DB状態
- 独立したReference Implementation
- 状態遷移ルール

---

# 20. 推奨する成果物構造

機能単位で、以下を中間成果物として管理する。

```text
feature/
├── specification/
│   ├── requirement.yaml
│   ├── behavior.yaml
│   ├── decision-table.yaml
│   ├── state-model.yaml
│   ├── contract.yaml
│   └── feature.feature
│
├── test-design/
│   ├── test-perspectives.yaml
│   ├── test-obligations.yaml
│   ├── test-level-assignment.yaml
│   └── test-cases.yaml
│
└── tests/
    ├── unit/
    ├── integration/
    ├── contract/
    ├── ui/
    └── e2e/
```

必ずしもファイルをこの構成に固定する必要はないが、**仕様・テスト設計・テストコードを別層として扱う**ことを推奨する。

---

# 21. 採用方針のまとめ

## 中核として採用

```text
Requirement
   ↓
Behavior
   ↓
Test Perspective
   ↓
Test Model
   ├─ Gherkin
   ├─ Decision Table
   ├─ State Model
   ├─ Contract
   └─ Property / Invariant
   ↓
Test Obligation
   ↓
Test Level Assignment
   ↓
Test Case
   ↓
Test Code
```

## テスト実装・検証の候補

```text
JUnit 5       : Unit / Integrationの基盤
AssertJ       : Assertion
jqwik         : Property-based Testing
Testcontainers: 実DB等を使ったIntegration
Pact          : Consumer Driven Contract Testing
Playwright    : UI / E2E
ArchUnit      : Architecture Test
PIT           : Mutation Testing
Coverage tool : 実行漏れ検出
```

## 過剰にしないもの

以下を一律に全機能へ適用しない。

- すべてのテストをGherkin化
- すべてのテストをE2E化
- すべてのUnit TestをProperty Test化
- 全条件の全組合せテスト
- すべての処理を1つの巨大なTest Methodに統合
- 1つの観点ごとに機械的に1テスト生成
- Codegenを全E2E作成の必須工程にする
- Coverage 100%を品質保証の最終基準にする

---

# 22. 最終的な判断基準

LLMが生成したテストについて、次の順序で評価する。

### ① 仕様上の保証対象が洗い出されているか

```text
Test Perspective / Test Model
```

### ② 具体的な保証義務になっているか

```text
Test Obligation
```

### ③ 適切なテストレベルに割り当てられているか

```text
Unit / Integration / UI / E2E
```

### ④ 重複を避けてTest Suiteが設計されているか

```text
複数観点 → 1 Test Case
```

が適切なら統合する。

### ⑤ Expected Resultが独立して妥当か

### ⑥ 実行によって保証されているか

### ⑦ Mutationによってテストの検出能力が確認できるか

### ⑧ Coverageに未実行部分がないか

この順序で評価する。

---

# 23. 目指すアーキテクチャ

最終的には、LLMを「テストコードをたくさん書くツール」としてではなく、**仕様からテスト設計を組み立て、機械的検証結果を受けて不足を補正するエンジン**として利用する。

```text
                         ┌──────────────┐
                         │ Requirements │
                         └──────┬───────┘
                                ↓
                    ┌───────────────────────┐
                    │ Test Perspective      │
                    └───────────┬───────────┘
                                ↓
              ┌─────────────────────────────────┐
              │ Test Model                      │
              │ Gherkin / Decision / State /   │
              │ Contract / Property             │
              └───────────────┬─────────────────┘
                              ↓
                    ┌──────────────────┐
                    │ Test Obligation  │
                    └────────┬─────────┘
                             ↓
                    ┌──────────────────┐
                    │ Test Level       │
                    └────────┬─────────┘
                             ↓
                    ┌──────────────────┐
                    │ Test Case Design │
                    └────────┬─────────┘
                             ↓
                    ┌──────────────────┐
                    │ LLM Code Gen     │
                    └────────┬─────────┘
                             ↓
       ┌─────────────────────┼──────────────────────┐
       ↓                     ↓                      ↓
     JUnit                Contract              Playwright
       │                     │                      │
       └─────────────────────┼──────────────────────┘
                             ↓
                ┌─────────────────────────┐
                │ Automated Validation   │
                │ Coverage / Mutation /  │
                │ Contract / Architecture│
                └────────────┬────────────┘
                             ↓
                    ┌──────────────────┐
                    │ Gap / Failure    │
                    │ Analysis by LLM  │
                    └────────┬─────────┘
                             ↓
                       Test Revision
```

この構造を基本アーキテクチャとする。

---

## 参考資料

- Playwright Browsers: https://playwright.dev/docs/browsers
- Playwright Test Generator: https://playwright.dev/docs/codegen
- Playwright CI: https://playwright.dev/docs/ci
- Cucumber Gherkin Reference: https://cucumber.io/docs/gherkin/reference/
- jqwik User Guide: https://jqwik.net/docs/current/user-guide
- PIT Mutation Testing: https://pitest.org/quickstart/basic_concepts/
- Pact Documentation: https://docs.pact.io/
- Testcontainers Java: https://java.testcontainers.org/
- ArchUnit User Guide: https://www.archunit.org/userguide/html/000_Index.html
