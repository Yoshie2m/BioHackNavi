# タスクと一次設計案

`CLAUDE.md` 第11章「開発の進め方」を、実行可能なタスクに分解したもの。
**一次案**であり、ドメインエキスパートのレビュー後に見直す。用語は `CLAUDE.md` 第2章に従う。

> 医学的な基準値・閾値（確実性の判定基準、適用条件の閾値、因果の確からしさの上限など）は
> 推測で決めず、`TODO(要確認)` として残す（`CLAUDE.md` 第12章）。

## 0. 初期スコープ

- 疾患：骨粗しょう症（1疾患）
- 栄養素：ビタミン D、カルシウム、ビタミン K（数栄養素）
- 目的：縦に一通り（domain → application → UI）通し、モデルの妥当性を検証する

## 1. タスク一覧

| ID | タスク | 対象コンテキスト | 依存 | 完了条件 |
| :-- | :-- | :-- | :-- | :-- |
| T0 | プロジェクト雛形（Vite + React + TS strict + Vitest） | 全体 | なし | `npm test` / `npm run lint` が通る |
| T1 | 依存ルールの機械検査（ESLint import 制約） | 全体 | T0 | 第5章違反がCIで失敗する |
| T2 | shared/kernel（Result 型、ID 型、日付型） | shared | T0 | 単体テスト付きで提供 |
| T3 | association の値オブジェクト | association | T2 | 不変条件のテストが通る |
| T4 | association の集約 `NutrientDiseaseAssociation` | association | T3 | 第6.4章の不変条件を満たす |
| T5 | ドメインサービス（Synthesis / Causality / Applicability） | association | T4, T7 | テスト駆動で実装 |
| T6 | application 層（ユースケース、リポジトリ IF、インメモリ実装） | association | T4 | インメモリで動作 |
| T7 | evidence の domain（Study / Evidence / EffectSize 等） | evidence | T2 | 単体テストが通る |
| T8 | catalog の domain と ConditionRef 変換（ACL） | catalog | T2 | Condition → ConditionRef 変換のテスト |
| T9 | 共通 UI（AssociationCard、免責表示、EvidenceLink） | shared/ui, association | T6 | 確実性が必ず表示されるテスト |
| T10 | 画面：栄養素詳細・疾患詳細・関連詳細・エビデンス詳細 | ui | T6, T9 | モックデータで導線が通る |
| T11 | 画面：用語集 | shared | T0 | 第2章の表と一致 |
| T12 | E2E（栄養素 → 関連 → エビデンス） | tests | T10 | Playwright が通る |
| T13 | profile / analysis（摂取量の突き合わせ） | profile, analysis | T5, T10 | 情報提供に限定された出力 |
| T14 | infrastructure（API / Zod / マッパー、外部標準 ACL） | 各コンテキスト | T6 | MSW でのテストが通る |
| T15 | ドメインエキスパートレビューと用語表の更新 | 全体 | 随時 | 第2章が最新 |

推奨順序：T0 → T1 → T2 → T3/T7 → T4 → T5 → T6 → T8 → T9 → T10/T11 → T12 → T13 → T14（T15 は随時）

## 2. 各タスクの一次設計案

### T0 / T1：雛形と依存ルール検査

- 構成は `CLAUDE.md` 第4章どおり。`src/contexts/<context>/{domain,application,infrastructure,ui}` と `index.ts`
- ESLint は `eslint-plugin-boundaries`（または `no-restricted-imports`）で以下を禁止する
  - `domain` から `react` / `@tanstack/*` / `zod` / ブラウザ API への import
  - 他コンテキストの `domain` 内部への import（`index.ts` 経由のみ可）
  - `ui` から `infrastructure` の DTO への import

### T2：shared/kernel

```ts
export type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };
export const ok = <T>(value: T): Result<T, never> => ({ ok: true, value });
export const err = <E>(error: E): Result<never, E> => ({ ok: false, error });
```

- ID 型はブランド型（例：`AssociationId`、`NutrientRef`、`ConditionRef`）で取り違えを防ぐ
- 日付型は ISO 8601 文字列のブランド型（`assessedAt` で使用）

### T3：association の値オブジェクト

| 値オブジェクト | 一次案 | 備考 |
| :-- | :-- | :-- |
| Direction | `'risk-reducing' \| 'risk-increasing' \| 'no-association' \| 'inconclusive'` | 第2章 |
| EvidenceGrade | `'high' \| 'moderate' \| 'low' \| 'very-low' \| 'insufficient'` | GRADE 準拠。`canBeAdvice` を持つ |
| CausalPlausibility | 段階（例：`'strong' \| 'moderate' \| 'weak' \| 'unknown'`） | `TODO(要確認)` 段階の定義と上限規則 |
| DoseRange | 下限・上限・単位（単位必須） | 範囲の検証をファクトリで行う |
| Applicability | 年齢範囲・性別・DoseRange | `TODO(要確認)` 具体的な閾値 |

- すべて `Readonly` + ファクトリ関数。不正値は `Result` で返す

### T4：集約 `NutrientDiseaseAssociation`

- 保持：`AssociationId`、`NutrientRef`、`ConditionRef`、`Direction`、`EvidenceGrade`、`CausalPlausibility`、`Applicability`、`EvidenceRef[]`、`assessedAt`
- 不変条件（第6.4章）とテストの対応

| 不変条件 | テスト名（案） |
| :-- | :-- |
| 確実性が不十分の関連は助言として出力できない | 確実性が不十分の関連は助言として出力されない |
| inconclusive のとき確実性を高にできない | 方向が結論不能の関連は確実性を高にできない |
| 観察研究のみの場合、因果の確からしさに上限 | 観察研究のみの関連は因果の確からしさが上限を超えない |
| 根拠（EvidenceRef）が 1 件以上必要 | 根拠のない関連は生成できない |

### T5：ドメインサービス

- `EvidenceSynthesisService`：`Evidence[]` → `{ direction, evidenceGrade }`。統合規則は `TODO(要確認)`（初期はプレースホルダー実装）
- `CausalityAssessor`：Bradford Hill の基準を入力に `CausalPlausibility` を返す。基準の重み付けは `TODO(要確認)`
- `ApplicabilityChecker`：`(Applicability, 利用者属性, NutrientIntake)` → 適用可否と理由。範囲外は「適用しない」

### T6：application 層

- ユースケース案：`GetAssociationsByNutrient`、`GetAssociationsByCondition`、`GetAssociationDetail`
- リポジトリ IF：`AssociationRepository`（`findById` / `findByNutrient` / `findByCondition`）
- インメモリ実装をテスト・モック画面で共用
- ui へは hooks 経由で公開（TanStack Query はここではなく ui 側のアダプタに置く）

### T7：evidence の domain

- 集約：`Study`（デザイン、サンプルサイズ、限界）、`Evidence`（`Study` への参照、`EffectSize`、`ConfidenceInterval`）
- `StudyDesign`：`'RCT' | 'cohort' | 'case-control' | ...`。観察研究判定（`isObservational`）を提供し、T4/T5 から利用する

### T8：catalog と ACL

- catalog の `Condition`（ICD / MeSH コード保持）→ association の `ConditionRef`（ID のみ）への変換関数を `infrastructure/mappers` に置く
- 初期はコード表をモックデータで持つ

### T9：共通 UI

- `<AssociationCard>`：Direction、EvidenceGrade、CausalPlausibility、Applicability、根拠リンクを必ず描画する（props で省略不可にする）
- `<DisclaimerNotice>`：情報提供であり医療判断の代替ではない旨。全分析画面で共通利用
- `<LastAssessedAt>`：最終評価日の表示
- UI 文言は因果断定表現を禁止。文言テストで担保する

### T10：画面とルーティング

- `CLAUDE.md` 第7.4章のパス（`/nutrients/:id`、`/conditions/:id`、`/associations/:id`、`/evidence/:id`）
- すべての関連表示から根拠エビデンスまで 2 クリック以内（第7.5章）
- ページコンポーネントはユースケース hooks を呼んで表示するだけ

### T13：profile / analysis

- `NutrientIntake`（量・単位・期間・申告精度）、単位変換は値オブジェクト内
- analysis は `ApplicabilityChecker` を用いて突き合わせ、結果は「情報提供」表現に限定
- 要配慮情報は localStorage に保存しない。ログ・アナリティクスに送らない（第9章）

### T14：infrastructure

- API レスポンスは Zod で検証し DTO として扱い、`mappers` でドメインモデルに変換
- 外部標準（ICD / MeSH / 食品成分表 / PubMed）のコードはこの層で内部 ID に変換
- テストは Vitest + MSW

## 3. 未決事項（要確認）

| # | 内容 | 確認先 |
| :-- | :-- | :-- |
| 1 | CausalPlausibility の段階定義と、観察研究のみの場合の上限 | 医師・研究者 |
| 2 | 複数エビデンスの統合規則（EvidenceSynthesisService） | 研究者 |
| 3 | 適用条件の具体的な閾値（年齢・性別・用量範囲） | 管理栄養士・医師 |
| 4 | 初期データの出典（骨粗しょう症 × ビタミン D 等）と評価時点 | 研究者 |
| 5 | SaMD 該当性のレビュー方針 | 法務・専門家 |
| 6 | バックエンド有無（保存はサーバー側が原則。API 仕様の前提） | 開発チーム |
