# タスクと一次設計案

`CLAUDE.md` 第11章「開発の進め方」を、実行可能なタスクに分解したもの。
**一次案**であり、ドメインエキスパートのレビュー後に見直す。用語は `CLAUDE.md` 第2章に従う。

> 医学的な基準値・閾値（確実性の判定基準、適用条件の閾値、因果の確からしさの上限など）は
> 推測で決めず、`TODO(要確認)` として残す（`CLAUDE.md` 第12章）。

## 0. 初期スコープ

- 疾患：骨粗しょう症（1疾患）
- 栄養素：ビタミン D、カルシウム、ビタミン K（数栄養素）
- 目的：縦に一通り（domain → application → UI）通し、モデルの妥当性を検証する

## 0.1 実行方針：フロントエンドのみで動作（バックエンドなし）

当面は静的ホスティング（例：GitHub Pages）だけで動くシステムとして実装する。
DDD の層構成は変えず、**infrastructure の実装だけを差し替え可能にしておく**ことで、将来 API 化できるようにする。

| 項目 | 方針 |
| :-- | :-- |
| 知識データ（関連・エビデンス・カタログ） | リポジトリ内の静的 JSON（`public/data/*.json`）として管理し、起動時に取得する |
| リポジトリ実装 | application の IF に対し `StaticJsonXxxRepository` を実装する。テスト用のインメモリ実装と並存 |
| データ検証 | 読み込み時に Zod で検証し、DTO → ドメインモデルへ変換（ACL）。加えてビルド前にデータ検証スクリプトを CI で実行 |
| サーバー状態 | TanStack Query をそのまま利用（静的 JSON の取得・キャッシュ） |
| ルーティング | 静的ホスティング対応（`createHashRouter`、または SPA フォールバック設定） |
| 外部標準・PubMed 等の API 連携 | 当面なし。ICD / MeSH コードは JSON に事前に埋め込む |
| 利用者プロファイル（摂取量など） | **ブラウザ内メモリのみ**（リロードで消える）。永続化・送信はしない（下記の注意参照） |
| 認証・ユーザー管理 | なし |
| 将来のバックエンド化 | repository 実装を HTTP 版に差し替えるだけで済む構成を維持する |

### 注意：`CLAUDE.md` 第9章との関係

第9章は「保存はサーバー側を原則、localStorage 等への平文保存は禁止」としている。
サーバーがない間は**利用者の健康情報を一切永続化しない**ことでこの制約を満たす。
ファイルへのエクスポート／インポートや localStorage 利用を入れる場合は、第9章の見直しと合意が必要（未決事項 #6）。

## 0.2 設計ドキュメントの構成（`docs/ddd/` に一元管理）

DDD の成果物を種類ごとにフォルダ分けし、`docs/ddd/` に集約する。コードの構成（`src/`）とは独立して管理する。

```
docs/
└─ ddd/
   ├─ README.md                      # 設計書の目次、読む順番、更新ルール
   ├─ 00-domain-vision.md            # ドメインビジョン、提供価値、スコープ外
   ├─ strategic/
   │   ├─ ubiquitous-language.md     # 用語集（全コンテキスト共通）
   │   ├─ subdomains.md              # コア／支援／汎用の分類と理由
   │   ├─ context-map.md             # コンテキスト間の関係（ACL、公開API等）
   │   └─ event-storming/
   │       └─ 2026-10-vitamin-d.md   # セッションごとの記録
   ├─ tactical/
   │   ├─ association.md             # 集約、値オブジェクト、不変条件、サービス
   │   ├─ evidence.md
   │   ├─ catalog.md
   │   ├─ profile.md
   │   └─ analysis.md
   ├─ policies/
   │   ├─ invariants.md              # 不変条件の一覧とテストへの対応表
   │   └─ regulatory-and-privacy.md  # 医療行為との境界、要配慮個人情報
   └─ adr/
       ├─ 0001-separate-association-and-causality.md
       └─ 0002-use-result-type.md    # 設計判断の記録（1判断1ファイル）
```

### 既存資料との対応（どこから何を移すか）

| 新規ドキュメント | 元になる記述 | 備考 |
| :-- | :-- | :-- |
| `00-domain-vision.md` | `CLAUDE.md` 第1章 | 提供価値・提供しないもの（スコープ外） |
| `strategic/ubiquitous-language.md` | `CLAUDE.md` 第2章 | **正（Single Source of Truth）をどちらに置くか要決定**（未決事項 #8） |
| `strategic/subdomains.md` | `CLAUDE.md` 第3章の「位置づけ」列 | コア／支援／汎用の分類理由を追記 |
| `strategic/context-map.md` | `CLAUDE.md` 第3章（コンテキスト間ルール）・第5章 | 関係の種類（ACL、公開 API）を図示 |
| `strategic/event-storming/…` | 新規 | 初回は「ビタミン D × 骨粗しょう症」 |
| `tactical/association.md` | `CLAUDE.md` 第6.1章、本書 T3〜T5 | 集約・値オブジェクト・サービス |
| `tactical/evidence.md` | `CLAUDE.md` 第6.2章、本書 T7 | |
| `tactical/catalog.md` | `CLAUDE.md` 第3章、本書 T8 | 静的 JSON の構造（0.1 節）を含む |
| `tactical/profile.md` / `analysis.md` | `CLAUDE.md` 第6.3章、本書 T13 | メモリ上のみ保持（0.1 節） |
| `policies/invariants.md` | `CLAUDE.md` 第6.4章、本書 T4 の対応表 | 不変条件 ⇔ テスト名の対応表。実装済みかを列で管理 |
| `policies/regulatory-and-privacy.md` | `CLAUDE.md` 第9章 | フロントエンドのみ時の読み替えを記載 |
| `adr/0001-separate-association-and-causality.md` | `CLAUDE.md` 第2章「関連と因果を混同しない」 | 判断理由・代替案・影響 |
| `adr/0002-use-result-type.md` | `CLAUDE.md` 第7.1章 | 例外ではなく `Result` を使う |
| `adr/0003-frontend-only-static-json.md`（追加提案） | 本書 0.1 節 | 当面はバックエンドなし |

### 運用ルール（`docs/ddd/README.md` に記載する）

- 読む順番：`00` → `strategic/` → `tactical/` → `policies/` → `adr/`
- 1 つの設計判断につき ADR を 1 ファイル作る（番号は連番、状態は `提案 / 採用 / 廃止`）。採用後は書き換えず、変更時は新しい ADR で置き換える
- 用語・モデルを変更したら、`ubiquitous-language.md` と該当する `tactical/*.md` を**同じ変更で**更新する（`CLAUDE.md` 第11章 6 の延長）
- 不変条件を追加したら `policies/invariants.md` に追記し、対応するテスト名を書く
- 医学的な判断で未確定のものは `TODO(要確認)` を残し、確認先を明記する
- `PROPOSAL.md` は初期案の記録として残し、内容が `docs/ddd/` に移った項目には移行先へのリンクを追記する

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
| T14 | infrastructure（静的 JSON ローダー / Zod / マッパー）とデータ検証スクリプト | 各コンテキスト | T6 | 不正な JSON を検出するテストが通る |
| T16 | 静的ホスティング対応（ルーティング、base パス、デプロイ設定） | app | T10 | 静的ホスティング上で全画面が表示される |
| T17 | 初期データ作成（骨粗しょう症 × 栄養素の JSON、評価時点つき） | catalog, association, evidence | T14 | ドメインエキスパート確認済み |
| T18 | （将来）HTTP 版リポジトリへの差し替え | 各コンテキスト | T14 | 現時点では対象外 |
| T15 | ドメインエキスパートレビューと用語表の更新 | 全体 | 随時 | 第2章が最新 |

推奨順序：T0 → T1 → T2 → T3/T7 → T4 → T5 → T6 → T8 → T9 → T10/T11 → T12 → T13 → T14（T15 は随時）

### 設計ドキュメント作成タスク（`docs/ddd/`）

実装タスク（T 系）と並行して進める。D 系の完了は、対応する T 系の着手前または完了時を目安にする。

| ID | タスク | 成果物 | 依存 | 完了条件 |
| :-- | :-- | :-- | :-- | :-- |
| D0 | 目次と更新ルール、ドメインビジョン | `README.md`、`00-domain-vision.md` | なし | 読む順番と更新ルールが書かれている |
| D1 | ユビキタス言語の移行 | `strategic/ubiquitous-language.md` | 未決 #8 | 正の置き場が決まり、重複がない |
| D2 | サブドメイン分類 | `strategic/subdomains.md` | D0 | コア／支援／汎用の理由が書かれている |
| D3 | コンテキストマップ | `strategic/context-map.md` | D2 | 全コンテキスト間の関係と ACL が図示されている |
| D4 | イベントストーミング（初回） | `strategic/event-storming/2026-10-vitamin-d.md` | D1 | ドメインエキスパートのレビューを経ている |
| D5 | association の戦術設計 | `tactical/association.md` | D1, D3 | 集約・値オブジェクト・サービスが T3〜T5 と一致 |
| D6 | evidence / catalog の戦術設計 | `tactical/evidence.md`、`catalog.md` | D5 | T7・T8 と一致 |
| D7 | profile / analysis の戦術設計 | `tactical/profile.md`、`analysis.md` | D5 | T13 と一致。メモリ上のみ保持を明記 |
| D8 | 不変条件とテストの対応表 | `policies/invariants.md` | D5 | 第6.4章の全項目にテスト名がある |
| D9 | 規制・プライバシー方針 | `policies/regulatory-and-privacy.md` | D0 | フロントエンドのみ時の読み替えを含む |
| D10 | ADR（0001〜0003） | `adr/0001〜0003-*.md` | D0 | 判断理由・代替案・影響が書かれている |

推奨順序：D0 → D1 → D2/D3 → D10 → D4 → D5 → D8 → D6/D7 → D9

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
- フロントエンドのみのため、プロファイル・摂取量は**メモリ上のみ**で保持し、永続化しない。localStorage に保存しない。ログ・アナリティクスに送らない（第9章）
- 状態は `useReducer` またはコンテキスト内の Provider で保持（グローバルストアにドメイン状態を置かない）

### T14：infrastructure（静的 JSON）

- `public/data/` に `nutrients.json`、`conditions.json`、`associations.json`、`evidence.json` を置く（コンテキストごとに分割）
- 読み込んだ JSON は Zod で検証し DTO として扱い、`mappers` でドメインモデルに変換
- JSON 内の ICD / MeSH コードはこの層で内部 ID に変換
- `scripts/validate-data.ts`：Zod スキーマで全 JSON を検証し、参照切れ（存在しない EvidenceRef など）も検出。CI で実行
- テストは Vitest（JSON フィクスチャ）。HTTP 通信がないため MSW は当面不要

### T16：静的ホスティング対応

- `createHashRouter` を採用（またはホスティング側の SPA フォールバック設定）
- Vite の `base` を設定可能にする（GitHub Pages のサブパス対応）
- デプロイは GitHub Actions で `build` → Pages へ公開（方式は未決）

### T17：初期データ

- 各レコードに `assessedAt` と出典（EvidenceRef）を必須とする
- 評価値（EvidenceGrade、CausalPlausibility など）は専門家確認までは `TODO(要確認)` のダミー値と明示し、画面にも「暫定データ」を表示する

## 3. 未決事項（要確認）

| # | 内容 | 確認先 |
| :-- | :-- | :-- |
| 1 | CausalPlausibility の段階定義と、観察研究のみの場合の上限 | 医師・研究者 |
| 2 | 複数エビデンスの統合規則（EvidenceSynthesisService） | 研究者 |
| 3 | 適用条件の具体的な閾値（年齢・性別・用量範囲） | 管理栄養士・医師 |
| 4 | 初期データの出典（骨粗しょう症 × ビタミン D 等）と評価時点 | 研究者 |
| 5 | SaMD 該当性のレビュー方針 | 法務・専門家 |
| 6 | サーバーなしでの利用者データの扱い（現案：永続化しない。エクスポート／インポートを許すか、第9章をどう読み替えるか） | 開発チーム・法務 |
| 7 | 静的ホスティング先とデプロイ方式（GitHub Pages など） | 開発チーム |
| 8 | ユビキタス言語の正をどこに置くか。案：`docs/ddd/strategic/ubiquitous-language.md` を正とし、`CLAUDE.md` 第2章はそこへのリンクと要約にする（`CLAUDE.md` の改訂が必要）。`/glossary` 画面のデータ元にもなる | 開発チーム |
| 9 | `docs/ddd/` の言語（日本語で統一する想定でよいか）と、図の記法（Mermaid など） | 開発チーム |
