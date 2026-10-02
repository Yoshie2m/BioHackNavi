# CLAUDE.md — 栄養素×疾患 関連分析サイト（React / DDD）

このファイルは、プロジェクトの設計方針とコーディング規約を AI／開発者に共有するためのものです。
実装・レビュー・リファクタリングの際は、必ずここに書かれたユビキタス言語、境界、依存ルールに従ってください。

---

## 1. プロジェクト概要

病気（疾患・病態）の原因と栄養素の関係を、エビデンスに基づいて閲覧・分析できるウェブサイト。

- 提供価値：栄養素と疾患の関連について「どの程度確からしいか」「どの条件で当てはまるか」「根拠は何か」を示す
- 提供しないもの：診断、治療方針の指示、個別の医療判断（第9章を参照）
- 開発手法：ドメイン駆動設計（DDD）。フロントエンドもドメインモデルを中心に構成する

### 技術スタック（前提。変更する場合は理由を残すこと）

| 領域 | 採用 |
| :-- | :-- |
| UI | React 18+ / TypeScript（strict） |
| ビルド | Vite |
| ルーティング | React Router |
| サーバー状態 | TanStack Query |
| バリデーション | Zod（外部入力の境界でのみ使用） |
| テスト | Vitest / React Testing Library / Playwright（E2E） |
| スタイル | Tailwind CSS（または CSS Modules） |
| 品質 | ESLint（依存ルール検査含む）/ Prettier |

---

## 2. ユビキタス言語

コード・UI文言・コミットメッセージ・ドキュメントで、以下の用語を**統一して**使うこと。
別の言い回し（例：relation、link を Association の代わりに使う）は禁止。

| 用語（日本語） | 識別子 | 定義 |
| :-- | :-- | :-- |
| 栄養素 | Nutrient | ビタミン、ミネラル、たんぱく質など、摂取対象として定義された物質 |
| 疾患 | Condition | 疾患・病態。外部標準（ICD / MeSH）のコードで識別する |
| 関連 | Association | 栄養素と疾患の間に研究で観察された結びつき。**因果を含意しない** |
| 因果の確からしさ | CausalPlausibility | 関連が因果であると言える程度の評価。関連とは別概念 |
| エビデンス | Evidence | 関連を支持／否定する個々の研究結果 |
| 研究 | Study | 論文などの一次情報（RCT、コホート、症例対照など） |
| エビデンス確実性 | EvidenceGrade | GRADE 等に基づく確実性の等級 |
| 効果量 | EffectSize | リスク比、オッズ比などの大きさ |
| 信頼区間 | ConfidenceInterval | 効果量の不確実性の範囲 |
| 適用条件 | Applicability | 関連が成り立つ対象（年齢・性別・用量範囲など） |
| 摂取量 | NutrientIntake | 量・単位・期間を必ず伴う値 |
| 摂取基準 | IntakeReference | 推奨量、耐容上限量など（日本人の食事摂取基準 等） |
| 方向 | Direction | risk-reducing / risk-increasing / no-association / inconclusive |

### 言語ルール

- **「関連」と「因果」を混同しない。** 型・コンポーネント名・UI文言のすべてで区別する
- UI 上は「〜を予防する」「〜の原因となる」のような**因果断定表現を避け**、確実性とセットで提示する
- 識別子は英語、UI 文言は日本語。対応表はこの章を正とする

---

## 3. 境界づけられたコンテキスト

フロントエンドは、以下のコンテキストを**機能単位のモジュール**として分離する。

| コンテキスト | ディレクトリ | 責務 | 位置づけ |
| :-- | :-- | :-- | :-- |
| 関連性 | `src/contexts/association` | 栄養素-疾患関連の表現・確実性・適用条件 | **コア** |
| エビデンス | `src/contexts/evidence` | 研究・エビデンス・確実性評価の表示 | コア寄り |
| カタログ | `src/contexts/catalog` | 栄養素・疾患・食品の定義と検索 | 支援 |
| 利用者プロファイル | `src/contexts/profile` | 食事記録・摂取量・属性（要配慮情報） | 支援 |
| 分析 | `src/contexts/analysis` | 摂取量と関連知識の突き合わせ結果の表示 | 支援 |
| 共通基盤 | `src/shared` | 認証、レイアウト、通知、汎用UI | 汎用 |

### コンテキスト間ルール

- 他コンテキストの **domain 内部を直接 import してはならない**
- 他コンテキストを使うときは、そのコンテキストの `index.ts`（公開API）経由のみ
- 用語の意味が異なる箇所（例：「疾患」）は、**腐敗防止層（ACL）** で変換する
  - カタログの Condition（MeSH/ICD コード）→ 関連性の ConditionRef（IDのみ）
  - 利用者プロファイルの既往歴 → 分析の ConditionRef

---

## 4. ディレクトリ構成

```
src/
├─ app/                       # アプリ起動、ルーティング、Provider
│   ├─ routes.tsx
│   └─ providers.tsx
├─ contexts/
│   ├─ association/           # コア
│   │   ├─ domain/            # 純粋な TypeScript。React・fetch に依存しない
│   │   │   ├─ model/         # 集約・エンティティ・値オブジェクト
│   │   │   ├─ services/      # ドメインサービス
│   │   │   └─ events/        # ドメインイベント（必要な場合）
│   │   ├─ application/       # ユースケース、リポジトリのインターフェース
│   │   ├─ infrastructure/    # API クライアント、DTO、マッパー（ACL）
│   │   ├─ ui/                # React コンポーネント、hooks、ページ
│   │   └─ index.ts           # 公開 API
│   ├─ evidence/
│   ├─ catalog/
│   ├─ profile/
│   └─ analysis/
├─ shared/
│   ├─ kernel/                # 共有カーネル（Result型、ID型、日付型など最小限）
│   ├─ ui/                    # デザインシステム（ドメイン非依存）
│   └─ lib/
└─ tests/                     # E2E・結合テスト
```

---

## 5. 依存ルール（厳守）

```
ui  →  application  →  domain
                          ↑
                   infrastructure（application のインターフェースを実装）
```

1. domain は **React、TanStack Query、fetch、Zod、ブラウザAPIに依存しない**
2. application は domain のみに依存する。リポジトリは**インターフェース**として定義する
3. infrastructure は API 通信と DTO ↔ ドメインモデルの変換を担当する
4. ui は application のユースケース（hooks 経由）を呼ぶ。**ui から DTO を直接扱わない**
5. 他コンテキストへの参照は公開 API 経由のみ
6. ESLint の import 制約ルールで上記を機械的に検査する（違反はCIで失敗）

---

## 6. ドメインモデル（コア）

### 6.1 関連性コンテキスト

**集約ルート：`NutrientDiseaseAssociation`**

- 識別子：`AssociationId`
- 保持するもの
  - `NutrientRef`、`ConditionRef`（ID参照のみ。他集約の実体を持たない）
  - `Direction`
  - `EvidenceGrade`（確実性）
  - `CausalPlausibility`（因果の確からしさ。関連の強さとは別に保持）
  - `Applicability`（年齢・性別・用量範囲など）
  - `EvidenceRef[]`（根拠エビデンスへの参照）
  - `assessedAt`（評価時点。エビデンスは更新されるため必須）

**値オブジェクト**：Direction、EvidenceGrade、CausalPlausibility、Applicability、DoseRange

**ドメインサービス**

- `EvidenceSynthesisService`：複数エビデンスから関連の方向・確実性を統合する
- `CausalityAssessor`：因果の確からしさを評価する（Bradford Hill の基準を参考）
- `ApplicabilityChecker`：利用者の属性・摂取量が適用条件に合致するか判定する

### 6.2 エビデンスコンテキスト

- 集約：Study、Evidence
- 値オブジェクト：EffectSize、ConfidenceInterval、StudyDesign（RCT / cohort / case-control / …）、SampleSize

### 6.3 利用者プロファイルコンテキスト

- 集約：HealthProfile、DietRecord
- 値オブジェクト：NutrientIntake（**量・単位・期間・申告精度を必ず持つ**）、LabValue
- 要配慮個人情報を含むため、保存・表示・送信に特別な制約がある（第9章）

### 6.4 不変条件（ドメイン層で強制する）

- 適用条件（用量範囲・年齢・性別）の**範囲外の利用者に、関連を適用しない**
- EvidenceGrade が「不十分」の関連は、**助言として出力できない**（「情報提供」のみ可）
- Direction が inconclusive のとき、確実性を「高」にできない
- NutrientIntake は単位なしで生成できない。単位変換は NutrientIntake 内で行う
- CausalPlausibility は、研究デザインが観察研究のみの場合、一定の上限を超えない
- 値オブジェクトは不変。コンストラクタではなく**ファクトリ関数**で検証し、不正値は Result で返す

---

## 7. フロントエンド実装ガイド

### 7.1 ドメイン層の書き方

```ts
// 例：値オブジェクトはファクトリ + 不変
export type EvidenceGrade = Readonly<{
  kind: 'EvidenceGrade';
  level: 'high' | 'moderate' | 'low' | 'very-low' | 'insufficient';
}>;

export const EvidenceGrade = {
  of(level: EvidenceGrade['level']): EvidenceGrade {
    return { kind: 'EvidenceGrade', level };
  },
  canBeAdvice(g: EvidenceGrade): boolean {
    return g.level !== 'insufficient';
  },
};
```

- class と関数のどちらでもよいが、プロジェクト内で統一する（デフォルトは**型 + 関数**）
- `any`、型アサーション（`as`）の乱用は禁止
- 例外ではなく `Result<T, E>` で失敗を表現する（`shared/kernel`）

### 7.2 インフラ層（ACL）

- API レスポンスは Zod で検証し、**DTO としてのみ扱う**
- DTO → ドメインモデルの変換は `infrastructure/mappers` に置く
- 外部標準（ICD / MeSH / 食品成分表）のコードは、この層で内部の ID に変換する
- 外部 API の仕様変更はこの層で吸収し、domain に波及させない

### 7.3 UI 層

- コンポーネントに**ドメインロジックを書かない**（判定・計算は domain / application へ）
- ページコンポーネントは、ユースケース hooks を呼んで表示するだけにする
- 確実性・因果の確からしさ・適用条件は、**関連の表示と必ずセットで表示**する（専用コンポーネント化）
  - 例：`<AssociationCard>` は Direction、EvidenceGrade、Applicability、根拠リンクを必ず描画する
- 状態管理：サーバー状態は TanStack Query、UI ローカル状態は useState/useReducer。グローバルストアはドメイン状態の置き場にしない

### 7.4 主な画面（初期スコープ）

| 画面 | パス | 内容 |
| :-- | :-- | :-- |
| 栄養素詳細 | `/nutrients/:id` | 関連する疾患の一覧と確実性 |
| 疾患詳細 | `/conditions/:id` | 関連する栄養素の一覧と確実性 |
| 関連詳細 | `/associations/:id` | 方向、効果量、確実性、適用条件、根拠エビデンス |
| エビデンス詳細 | `/evidence/:id` | 研究の概要、デザイン、限界 |
| 摂取量の分析 | `/analysis` | 入力した摂取量と関連知識の突き合わせ（情報提供） |
| 用語集 | `/glossary` | ユビキタス言語の一般向け解説 |

### 7.5 説明可能性

すべての分析結果・関連表示から、**根拠エビデンスまで 2 クリック以内に辿れる**こと。
画面上に出す主張は、必ず EvidenceRef を持つデータからのみ生成する。

---

## 8. テスト方針

| 層 | 方法 | 方針 |
| :-- | :-- | :-- |
| domain | Vitest（単体） | **最優先。** 不変条件・ドメインサービスをテスト駆動で実装。React 不要 |
| application | Vitest | リポジトリをインメモリ実装に差し替えてユースケースをテスト |
| infrastructure | Vitest + MSW | マッパー、Zod 検証、外部 API 変更への耐性 |
| ui | React Testing Library | 「確実性が必ず表示される」「因果断定文言が出ない」等をテスト |
| E2E | Playwright | 主要画面の導線（栄養素 → 関連 → エビデンス） |

- 第6.4章の不変条件は、**1つ以上のテストで必ず担保**する
- テスト名はユビキタス言語で書く（例：確実性が不十分の関連は助言として出力されない）

---

## 9. 規制・プライバシー上の制約（設計に反映する）

1. **医療行為・診断支援との境界**
   - 出力は「情報提供」に限定する。診断、治療、投薬、サプリ摂取の指示を示唆する文言は生成しない
   - プログラム医療機器（SaMD）該当性は、機能追加のたびにレビューする（法務・専門家確認）
   - すべての分析結果画面に、情報提供であり医療判断の代替ではない旨を表示する
2. **要配慮個人情報**
   - 利用者プロファイルは独立コンテキストとして扱い、他コンテキストへは必要最小限の値のみ渡す
   - ログ・エラー報告・分析ツール（アナリティクス）に健康情報を送らない
   - 保存はサーバー側を原則とし、localStorage 等への平文保存は禁止
3. **知識の時間変化**
   - 関連・エビデンスには評価時点（assessedAt）を持たせ、画面にも「最終評価日」を表示する
   - 過去の分析結果は、当時の根拠バージョンを参照できるようにする
4. **免責・注意表示は UI コンポーネントとして共通化**し、画面ごとに個別実装しない

---

## 10. 外部標準との接続（ACL の背後で）

| 対象 | 標準 |
| :-- | :-- |
| 疾患 | ICD-10/11、MeSH |
| 食品成分 | 日本食品標準成分表、USDA FoodData Central |
| 摂取基準 | 日本人の食事摂取基準 |
| 医療データ連携 | HL7 FHIR |
| 論文 | PubMed API |

外部標準のコードや構造は、infrastructure で内部モデルに変換する。ドメイン層では外部コードを直接扱わない。

---

## 11. 開発の進め方

1. **最初のスコープは狭く**：「ビタミン D と骨粗しょう症」など、1 疾患 × 数栄養素に限定する
2. コア（association）の domain 層を、テスト駆動で先に作る（UI なし）
3. モックデータ（インメモリのリポジトリ）で画面を通す
4. 外部 API・外部標準との接続（ACL）を後から実装する
5. ドメインエキスパート（管理栄養士・医師・研究者）のレビューで、用語とモデルを継続的に修正する
6. 用語・モデルを変更したら、**第2章のユビキタス言語表を同時に更新**する

---

## 12. AI への指示（このファイルを読み込んだ場合）

- 実装前に、対象がどのコンテキストに属するかを明示する
- 新しい概念を導入する際は、第2章の用語に既存の言葉がないか確認し、なければ追加を提案する
- 第5章の依存ルールに違反するコードは書かない。必要なら構成を提案する
- 「関連」を「因果」として表示・命名するコードは書かない
- 不明な医学的判断（確実性の基準、適用条件の閾値など）は、**推測で実装せずプレースホルダーとして残し、確認を求める**
- 新しい不変条件を実装したら、対応するテストを同時に追加する
