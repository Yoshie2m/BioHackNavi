# TASKS

進捗はチェックボックスで管理する（`[ ]` 未着手 / `[x]` 完了）。
当面はフロントエンドのみ（バックエンドなし）で動作させる。
詳細な設計案は `PROPOSAL.md` を参照。

## 準備
- [ ] T0 プロジェクト雛形（Vite + React + TS strict + Vitest）
- [ ] T1 依存ルールの機械検査（ESLint import 制約）
- [ ] T2 shared/kernel（Result 型、ID 型、日付型）

## コア（association / evidence / catalog）
- [ ] T3 association の値オブジェクト
- [ ] T4 集約 NutrientDiseaseAssociation
- [ ] T5 ドメインサービス（Synthesis / Causality / Applicability）
- [ ] T6 application 層（ユースケース、インメモリ実装）
- [ ] T7 evidence の domain
- [ ] T8 catalog の domain と ConditionRef 変換（ACL）

## UI
- [ ] T9 共通 UI（AssociationCard、免責表示、EvidenceLink）
- [ ] T10 画面：栄養素・疾患・関連・エビデンス詳細
- [ ] T11 画面：用語集
- [ ] T12 E2E（栄養素 → 関連 → エビデンス）

## 後続
- [ ] T13 profile / analysis（摂取量の突き合わせ）
- [ ] T14 infrastructure（静的 JSON ローダー / Zod / データ検証スクリプト）
- [ ] T15 ドメインエキスパートレビューと用語表の更新
- [ ] T16 静的ホスティング対応（HashRouter、base パス、デプロイ設定）
- [ ] T17 初期データ作成（骨粗しょう症 × 栄養素の JSON）
- [ ] T18 （将来）HTTP 版リポジトリへの差し替え ※現時点では対象外
