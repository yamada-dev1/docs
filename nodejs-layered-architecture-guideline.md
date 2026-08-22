# Node.jsバックエンド向け・実用重視のレイヤードアーキテクチャ設計ガイドライン

## 1. 基本構想

本ガイドラインでは、業務システムのNode.jsバックエンドを対象として、以下の考え方を基本構想とする。

- **Modular Monolith**を基本とし、機能単位でモジュール化する。
- レイヤー構造は、**Pragmatic Clean Architecture / Hexagonal Architectureの原則を必要な範囲で利用する**。
- Clean Architectureを教科書どおり全面適用するのではなく、**依存方向・責務分離という原則を採用し、レイヤーやPortを必要な場合だけ設ける**。
- 言語・フレームワークによって基本原則を変えない。Java、TypeScript/Node.js等でも原則は同じであり、実装手段や記述量だけを適宜変える。
- **疎結合そのものを目的にしない**。保守性、テスト容易性、変更影響範囲の制御といった具体的な目的がある場合に疎結合化する。
- DTO、Entity、Mapper、interfaceなどを形式的に増やさない。
- 単純CRUDと複雑な業務処理を同じ構造に無理に統一しない。

基本的なイメージは次のとおり。

```text
Feature Module
│
├─ Presentation
│
├─ Application       ← 必要なユースケースがある場合
│
├─ Domain            ← 複雑な業務ルールがある場合
│
└─ Infrastructure
    ├─ Repository
    ├─ External API Adapter
    └─ DB / ORM / SDK
```

重要なのは、**すべての機能がPresentation → Application → Domain → Infrastructureの全層を通る必要はない**という点である。

---

# 2. 主目的

本構成の主目的は、単純な「疎結合化」ではない。

### 2.1 業務ロジックをHTTP・DB・外部サービスから分離する

例えば、

```text
HTTPの都合
DBの都合
外部APIの都合
```

によって、

```text
注文確定のルール
在庫引当のルール
配送条件
```

といった業務ロジックが変わることを防ぐ。

### 2.2 テストしやすい構造にする

特に、

- 業務ルールのUnit Test
- ApplicationのUnit Test
- RepositoryのIntegration Test
- APIのIntegration / E2E Test

を適切に分離し、テスト速度と信頼性を両立させる。

### 2.3 変更影響範囲を限定する

例えばMySQLから別DBへ変更しても、業務ロジックまで変更する必要がない構造を目指す。

### 2.4 過剰設計を防ぐ

以下を目的化しない。

- interfaceを増やす
- DIコンテナを利用する
- DTOを分ける
- Entityを作る
- Domain層を作る
- Application Serviceを作る
- Repositoryを抽象化する

これらはすべて**手段**であり、目的ではない。

---

# 3. 基本的な依存方向

原則として、依存方向は以下とする。

```text
Presentation
    ↓
Application
    ↓
Domain
```

Infrastructureは外側の技術実装として扱う。

```text
Application ─────→ Port
                       ↑
                Infrastructure
```

ただし、これは「全機能で必ずこの経路を通す」という意味ではない。

単純CRUDなら、

```text
Presentation
    ↓
Repository
    ↓
DB
```

でもよい。

複雑な業務処理なら、

```text
Presentation
    ↓
Application
    ↓
Domain
    ↓
Repository / Port
    ↓
DB / External System
```

となる。

---

# 4. モジュール構成

全システムをレイヤーごとに分けるのではなく、**まず機能単位で分ける**。

推奨例：

```text
src/
├─ modules/
│  ├─ order/
│  │  ├─ presentation/
│  │  ├─ application/
│  │  ├─ domain/
│  │  └─ infrastructure/
│  │
│  ├─ inventory/
│  │  ├─ presentation/
│  │  ├─ application/
│  │  ├─ domain/
│  │  └─ infrastructure/
│  │
│  └─ shipment/
│
├─ shared/
│  ├─ logging/
│  ├─ auth/
│  ├─ config/
│  └─ db/
│
└─ app/
```

機能ごとに閉じ込めることで、システム規模が大きくなっても関連コードを追いやすくする。

`shared`には何でも入れない。共通化した方が本当に保守しやすいものだけを置く。

---

# 5. 設ける層と役割

## 5.1 Presentation

### 役割

HTTP/APIなど、外部からの入口を担当する。

主な責務：

- HTTP Requestの受け取り
- Path / Query / Header / Bodyの取得
- 入力形式のValidation
- 認証・認可の入口処理
- Applicationへの引き渡し
- HTTP Status / Response形式への変換
- HTTP固有のエラー処理

### 原則

**業務ルールを置かない。**

例えば、

```ts
if (order.status === "CANCELLED") {
    ...
}
```

のような注文状態に関する業務判断はControllerに置かない。

### Controllerの行数

「1メソッド10行以内」のような機械的ルールは設けない。

判断基準は行数ではなく、

> **HTTPに関する責務だけを担当しているか**

とする。

その結果として、多くのControllerは自然に短くなる。

---

# 6. Application層

## 6.1 役割

Applicationは、

> **システムが提供する意味のあるユースケースを実行・調整する**

役割を担う。

例えば、

- 注文を確定する
- 注文をキャンセルする
- 在庫を引き当てる
- 出荷を確定する
- 請求を確定する

など。

Applicationは、複数のRepositoryや外部サービスを組み合わせて処理を進めたり、トランザクション境界を管理したりする。

### 例

```text
ConfirmOrder
    │
    ├─ OrderRepository
    ├─ InventoryRepository
    ├─ PaymentGateway
    │
    └─ Domain
```

## 6.2 Applicationを設けないケース

単純なCRUD・単純検索などで、

```text
Controller
    ↓
Repository
    ↓
DB
```

で十分なら、形式的な

```text
GetUser
GetProduct
FindCustomer
```

などを作らない。

次のような処理を、

```ts
const user = await userRepository.findById(id);
return user;
```

というだけのApplication Serviceにすることには、必ずしも意味がない。

---

# 7. Domain層

## 7.1 役割

**業務ルールそのものを表現する。**

例えば、

- 注文確定可能条件
- キャンセル可能条件
- 割引計算
- 在庫引当ルール
- 状態遷移
- 金額計算
- 業務上の不変条件

など。

## 7.2 Domainを設ける基準

「Domain-Driven Designだから必ずDomain層を作る」のではなく、

> **業務ルールが複雑で、独立して表現・テストする価値があるか**

で判断する。

単純なCRUDではDomainが存在しなくてもよい。

---

# 8. Repository

## 8.1 基本原則

**DBアクセスはRepositoryに閉じ込めることを原則とする。**

すなわち、業務ロジックが直接、

- SQL
- ORM
- DB Client
- Connection
- Transaction API

などを扱わない。

基本形：

```text
Application / Domain
        ↓
    Repository
        ↓
  ORM / SQL / DB
```

これにより、永続化の詳細を業務ロジックから分離する。

## 8.2 Repositoryを設ける理由

Repositoryは「疎結合のため」だけに存在するものではない。

主な目的は、

- DBアクセスを集約する
- SQL / ORMを業務ロジックから隠す
- 永続化処理をまとめる
- DB関連のテスト境界を明確にする
- トランザクションやクエリの責務を整理する

ことである。

## 8.3 Repositoryのinterfaceは原則必須ではない

以下は別問題として扱う。

```text
DBアクセスをRepositoryに集約する
        ↓
原則

Repositoryをinterface経由にする
        ↓
必要な場合だけ
```

interfaceを導入する具体的な判断基準は次のとおり。

### interfaceを設ける具体的なケース

#### ケース1：Application / Domainから具体的実装を切り離す必要がある

```text
Application
    ↓
OrderRepository    ← Port
    ↑
SqlOrderRepository
```

DB実装をApplicationから独立させることに明確な価値がある場合。

#### ケース2：複数実装を切り替える

```text
OrderRepository
    ↑
 ┌──┴────────────┐
 │               │
MySQL          PostgreSQL
```

あるいは、

```text
RealRepository
FakeRepository
```

など。

#### ケース3：DBなしでApplication / Domainを大量にUnit Testしたい

例えば複雑な業務処理について、

```text
数百～数千件のテストケース
```

を高速に実行したい場合、Fake / Stub実装に差し替える意味がある。

ただし、**「テストしたい」という理由だけで全Repositoryをinterface化することは避ける**。

単純CRUDのRepositoryは、実DBを使ったIntegration Testで十分な場合が多い。

### interfaceを設けないケース

```text
Controller
    ↓
UserRepository
    ↓
MySQL
```

のような単純CRUDで、

- 実装が1つ
- DB切替の必要性がない
- Application / Domainの独立性が特に問題にならない
- RepositoryをIntegration Testすれば十分

なら、具体クラスだけでよい。

---

# 9. DomainからDBを取得する必要がある場合

原則として、

```text
Application
    ↓
Repository
    ↓
必要な情報を取得
    ↓
Domain
```

を第一候補とする。

例えば注文確定の判定に、

- 注文情報
- 顧客情報
- 在庫情報

が必要なら、Applicationが必要情報を集めてDomainに渡す。

```text
Application
 ├─ OrderRepository
 ├─ CustomerRepository
 └─ InventoryRepository
           ↓
        Domain
           ↓
    業務ルール判定
```

ただし、Domain上のルールを成立させるために外部情報への問い合わせ自体が業務上不可分である場合は、DomainからPortを介して参照する設計も認める。

重要なのは、

```text
Domain → MySQL
```

ではなく、

```text
Domain → 業務上意味のあるProvider / Port
```

とすることである。

---

# 10. RepositoryだけですべてのDBアクセスを表現しない

Repositoryを「万能CRUD窓口」にしない。

例えば、

```text
UserRepository
  findById
  findAll
  save
  update
  delete
  count
  search
  ...
```

という巨大なRepositoryにする必要はない。

また、大量データ集計や帳票・検索など、Entityの取得を目的としない処理については、

```text
QueryService
ReadModel
ReportQuery
```

などの専用アクセス方式を使ってよい。

つまり、

> **DBアクセスをRepositoryに閉じ込める**

ことと、

> **すべてのDBアクセスをEntity Repositoryに押し込む**

ことは同義ではない。

---

# 11. DTO / Entity / Mapper

## 原則

**データを移すだけの型を増やさない。**

避けたい例：

```text
RequestDto
    ↓
CommandDto
    ↓
DomainDto
    ↓
DomainEntity
    ↓
PersistenceEntity
    ↓
ResponseDto
```

すべての境界で詰め替える必要はない。

## 分離する判断

以下のような理由がある場合にのみ分離する。

- API公開形式と内部モデルを独立させたい
- DB構造を外部へ漏らしたくない
- 入出力モデルと業務モデルの意味が明確に異なる
- セキュリティ上、公開フィールドを厳密に制限する
- DBモデル変更の影響をAPIから切り離す

**「レイヤーが違うからDTOも別物にする」という理由だけでは分離しない。**

---

# 12. テスト自動化を中心にした設計

テスト容易性は重要な設計基準とするが、

> **テストのために疎結合化しすぎない**

ことを原則とする。

推奨するテストの役割分担：

```text
Unit Test
  ↓
Domain / Application
  ↓
高速・大量

Integration Test
  ↓
Repository / DB / External Adapter
  ↓
実際の技術要素を使用

E2E Test
  ↓
API / Browser / System全体
  ↓
主要シナリオを確認
```

特にDBアクセスについて、

> 「Unit TestだからRepositoryを全部Mockする」

という方針にはしない。

実DBを使ったIntegration Testの方が、SQL・ORM・制約・JOIN等を含めて実態に近い検証ができる。

---

# 13. 単純CRUDと複雑な処理を分ける

## 単純CRUD

```text
GET /users/123

Controller
    ↓
UserRepository
    ↓
DB
```

これで十分。

## ユースケースあり

```text
POST /orders/123/confirm

Controller
    ↓
ConfirmOrder
    ↓
OrderRepository
    ↓
DB
```

## 複雑な業務ルールあり

```text
Controller
    ↓
ConfirmOrder
    ↓
Domain
    ├─ Order
    ├─ OrderPolicy
    └─ ...
    ↓
Repository / Gateway
    ↓
DB / External System
```

この「必要に応じて深くなる構造」を許容する。

---

# 14. 制限・制約・禁止事項

## 14.1 行数規約をアーキテクチャ規約にしない

例えば、

- Controllerは10行以内
- Serviceは100行以内

などを絶対ルールにしない。

判断基準は**責務**。

## 14.2 すべてをDomainにしない

単純CRUDまで、

```text
Entity
ValueObject
DomainService
DomainEvent
```

などに分割しない。

## 14.3 すべてをApplication Serviceにしない

単なるRepository呼び出しの中継なら、Application層を設けない。

## 14.4 すべてをinterface化しない

将来差し替えるかもしれない、だけでは理由として弱い。

## 14.5 DTO変換を機械的に増やさない

境界に意味がある場合だけ分離する。

## 14.6 DIコンテナを目的化しない

DIが必要なら利用する。

DIコンテナ導入自体をアーキテクチャの目標にしない。

## 14.7 Microservices化を前提にしない

まずModular Monolithとし、実際に分離する必要が発生したモジュールだけ将来分離できる構造を目指す。

---

# 15. 実装時の判断フロー

新しい機能を追加するときは、以下の順で判断する。

### Step 1：単純CRUDか？

Yesなら、

```text
Controller
  ↓
Repository
  ↓
DB
```

から始める。

### Step 2：意味のあるユースケースがあるか？

例えば、

```text
注文確定
注文キャンセル
在庫引当
出荷確定
```

など。

YesならApplicationを設ける。

### Step 3：複雑な業務ルールがあるか？

YesならDomainを設ける。

### Step 4：外部技術との依存をApplication / Domainから分離する必要があるか？

YesならPort / interfaceを設ける。

### Step 5：単純なデータ変換だけで型を分割していないか？

不要なDTO / Mapperを削る。

### Step 6：その抽象化は具体的な問題を解決しているか？

「将来のため」「Clean Architectureだから」だけなら再検討する。

---

# 16. 設計レビュー時の確認事項

### 責務

- Controllerに業務ルールがないか
- Applicationがユースケースの実行・調整を担当しているか
- Domainに業務ルールが集約されているか
- RepositoryにSQL / ORM処理が閉じているか

### 依存

- 業務ロジックがDB・HTTP・外部APIへ直接依存していないか
- interface / Portを使う理由が明確か
- Portが単なる形式的なinterfaceになっていないか

### 複雑性

- 単純CRUDに不要な層を追加していないか
- DTO / Mapperが過剰でないか
- Repositoryが巨大化していないか
- 共通化しすぎて機能間の境界が曖昧になっていないか

### テスト

- Domain / Applicationの重要ロジックを高速にUnit Testできるか
- DB処理をIntegration Testできるか
- 主要な業務シナリオをE2Eで確認できるか
- Mockが多すぎて実装詳細に依存したテストになっていないか

---

# 17. 最終的な基本ルール

本ガイドラインの核は、次のとおり。

```text
1. 機能単位でモジュール化する
2. HTTPと業務ロジックを分離する
3. DBアクセスはRepository等の永続化境界に閉じ込める
4. 意味のあるユースケースがある場合にApplicationを設ける
5. 複雑な業務ルールがある場合にDomainを設ける
6. 依存方向を逆転する明確な理由がある場合にPort/interfaceを設ける
7. DTO・Mapper・Entityを必要以上に増やさない
8. 単純CRUDを複雑なアーキテクチャに合わせない
9. テスト容易性を重視するが、Mockやinterfaceの量を目的にしない
10. Clean Architectureの「形」ではなく、責務分離と依存方向という「原則」を採用する
```

最終的には、

> **「最初から全層を作る」のではなく、「必要な責務が発生した時点で、その責務を適切な層へ分離する」**

ことを中心原則とする。

これにより、単純なCRUDは単純なまま維持しながら、複雑な業務処理だけをApplication・Domain・Portによって段階的に構造化できる。

また、この方針はNode.js固有ではなく、Java、C#、Goなどにも基本的に適用できる。言語・フレームワークによって変えるのは主にDI、型、ORM、モジュール管理などの**実装手段**であり、責務分離そのものではない。
