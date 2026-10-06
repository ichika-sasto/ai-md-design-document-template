---
status: closed
priority:
issue_date: 2026-09-15
start_date:
due_date:
end_date:
version:
assignee:
parent:
labels:
  - "adr"
created_at: "2026-09-15 16:50:34"
updated_at: "2026-09-15 18:42:08"
---

# フロントエンドのグリッド（テーブル）実装方式

- 決定日: 2026-09-15
- 状態: 承認

## 背景

要望「フロントエンドのグリッド（テーブル）コンポーネント整備」で、表形式データの表示・
インライン編集・行内ボタン配置に使う標準部品を用意することになった。フォーム
（[016_frontend_form_architecture](016_frontend_form_architecture.md)）は既に vee-validate +
shadcn-vue で部品化されているが、グリッドに使えるライブラリはこれまで 1 つも入っていない。

SIer 案件ではグリッドのインライン編集が要件になることがあり、行内に操作ボタンを置く要求も
出る。この移行にあたって、後から「なぜこの構成か」を再調査させないために残しておくべき
決定が 3 つある。

- グリッド用ライブラリの**選定**
- 選んだライブラリへの依存を**どこに閉じ込めるか**（Enterprise 版への切り替えやすさに影響する）
- モジュール登録とテーマの**当て方**

## 選択肢

### 決定 1: グリッドライブラリの選定

#### 案 A: ヘッドレス — `@tanstack/vue-table`（MIT）＋ shadcn-vue の `table` プリミティブ

利点 /
- 描画は自分たちの Vue テンプレートになるので、セルに既存の `components/form/TextField.vue` を
  そのまま挿せる。インライン編集を vee-validate + zod の既存作法に載せやすい
- Tailwind / shadcn-vue のデザイントークンとそのまま馴染む
- 依存は 1 パッケージだけ増える

欠点 /
- 列固定・セル範囲選択・Excel 風のコピー＆ペーストのような「グリッドらしい」高度な機能は
  すべて自前実装になる。SIer 案件で将来こうした要件が出た場合、作り込んだ分を作り直すことになる

#### 案 B: フル機能 — PrimeVue DataTable（Apache-2.0）

利点 /
- セル編集・行編集・ページング・ソート・フィルタ・列固定・仮想スクロールが標準装備で、
  SIer 要件に短期間で届く

欠点 /
- shadcn-vue とは別のデザインシステム・CSS 体系がもう 1 つ入り、フォームの作法（vee-validate）
  とグリッドの編集作法（PrimeVue 独自）が二重になる

#### 案 C: フル機能 — Tabulator（MIT）

利点 /
- フレームワーク非依存のコアで、Vue3 との結合はごく薄いラッパーで済む
- セル編集（input / select / checkbox / date 等）・行内フォーマッタ（ボタン描画）・
  ソート・フィルタ・ページング・仮想スクロールが標準で全部入りで、AG Grid のように
  機能が Enterprise 版に分かれていない

欠点 /
- 独自の CSS テーマを持ち、shadcn-vue / Tailwind のデザイントークンとは別に見た目を
  合わせ込む作業が要る（案 B と同じ課題）
- セルの検証は Tabulator 独自の `validator` 設定で行う仕組みで、既存の vee-validate + zod の
  フォーム作法とは別系統になる

#### 案 D: フル機能 — AG Grid Community（MIT。ただし列固定・行グループ化・Excel 風のセル範囲選択 /
コピペ等の一部機能は Enterprise 版が必要）

利点 /
- Excel 的要件（セル編集・行内ボタン・ソート・フィルタ・ページング）への到達度が高く、
  Community 版（無料・MIT）だけで今回の MVP は完結する
- 将来、列固定・行グループ化・Excel 風のセル範囲選択のような高度な要件が出たとき、
  同じライブラリの Enterprise 版へライセンス購入だけで切り替えられる
  （ヘッドレス案のように自前実装を作り直す必要がない）

欠点 /
- バンドルサイズが大きい（数百 KB 規模）
- 独自スタイルを持ち、機能ごとに「これは Community か Enterprise か」を気にする運用コストが残る

#### 案 E: ライブラリを入れず自前で部品化する

利点 /
- 依存ゼロ。既存 ADR 016 の「依存を増やさず自前アダプタで繋ぐ」という思想に最も忠実

欠点 /
- 並べ替え・列定義・インライン編集の状態管理・行内ボタンの出し分けを全部自分で持つことになる。
  「ライブラリを選定する」という要望の主旨自体を満たさない

### 決定 2: AG Grid への依存を閉じ込める場所（コンポーネント構成）

#### 案 A: `frontend/src/components/grid/` にラッパーコンポーネント一式を作り、AG Grid への
参照をここへ閉じる。呼び出し側（`components/sample/` 等）は `@/components/grid/...` しか
import しない

利点 /
- 将来 AG Grid Enterprise へ切り替えるとき（`ModuleRegistry.registerModules` に
  `AllEnterpriseModule` を足し、`LicenseManager.setLicenseKey` を呼ぶ）、変更箇所が
  `components/grid/` の数ファイルだけで済む
- `components/form/` が vee-validate + shadcn-vue を包んでいるのと同じ構造になり、
  既存の層の分け方（ADR 016 決定 2）をそのまま踏襲できる

欠点 /
- ラッパー層が 1 つ増える分、初見のコードリーディングコストがわずかに増える

#### 案 B: 呼び出し側（各機能の `components/<機能>/`）が `ag-grid-community` / `ag-grid-vue3` を
直接 import する

利点 /
- ラッパー層を介さない分、実装ファイル数が少なくて済む

欠点 /
- AG Grid への参照がリポジトリ全体に散らばり、Enterprise へ切り替えるときの変更箇所が
  読めなくなる。ライブラリ自体を差し替える判断も難しくなる

### 決定 3: モジュール登録とテーマの当て方

AG Grid v33 以降は `ModuleRegistry.registerModules(...)` の呼び出しが必須（呼ばないと機能が
黙って効かない）。テーマは Theming API（`theme` グリッドオプション、CSS の import 不要）と、
レガシー CSS テーマ（`ag-grid.css` + `ag-theme-*.css` の import）の 2 通りがある。

#### 案 A: `AllCommunityModule` を一括登録する。テーマは Theming API を使う

利点 /
- 登録漏れ（機能を使ったのに動かない実行時エラー）を避けられる。登録は
  `components/grid/grid-setup.ts` の 1 行に閉じる
- Theming API なら CSS の import が不要で、AG Grid への参照を `components/grid/` に
  完全に閉じられる（レガシー CSS テーマは `main.ts` か `assets/main.css` での import が要り、
  決定 2 の「参照を 1 か所に閉じる」を満たせなくなる）
- `themeQuartz.withParams({ borderColor: 'var(--border)', ... })` のように、Tailwind の
  CSS 変数をそのままテーマパラメータへ渡せる

欠点 /
- 使わない機能のモジュールまで登録され、バンドルがわずかに大きくなる

#### 案 B: 使う機能のモジュールだけ個別登録する。テーマはレガシー CSS テーマを使う

利点 /
- バンドルサイズを絞れる
- レガシー CSS テーマは AG Grid のバージョンを問わず情報が多い

欠点 /
- 機能を足すたびにモジュール登録を見直す運用になり、登録漏れのリスクが常に残る
- CSS の import 元が `components/grid/` の外（`main.ts` 等）に必要になり、決定 2 の
  「参照を 1 か所に閉じる」方針と相性が悪い

## 決定

**採るのは、決定 1 = 案 D（AG Grid Community）、決定 2 = 案 A（`components/grid/` に閉じる）、
決定 3 = 案 A（一括登録 + Theming API）。**

理由:

- **決定 1**: ヘッドレス案（A）は既存フォーム層をそのまま使える利点が大きいが、SIer 案件で
  想定される「グリッドの編集が必須」「Excel 的な操作性」という将来要件に対して、
  自前実装を作り直すコストが読めない。AG Grid は Community 版だけで今回の MVP
  （セル編集・行内ボタン・ソート・ページング）に届き、かつ Enterprise 版へライセンス購入のみで
  段階的に拡張できる点を優先した。PrimeVue（B）・Tabulator（C）は完成度で近い水準にあるが、
  いずれも独自のデザインシステム・検証作法を持ち込み、AG Grid ほど「今後の拡張が同一ライブラリ内で
  完結する」明確さが無いため採らなかった。自前実装（E）は要望の主旨（ライブラリ選定）を
  満たさない
- **決定 2**: 決定 1 で「Enterprise へ段階的に拡張できること」を選定理由にした以上、
  実際に 1 箇所の変更で切り替えられる構成にしておかないと理由が成立しない。ADR 016 決定 2 で
  確立した「独自ラップ部品は専用フォルダへ」という方針とも一致する
- **決定 3**: 登録漏れは実行時にしか表面化せず、ラッパー 1 か所に閉じる方針（決定 2）と
  個別登録は相性が悪い。バンドル削減が必要になったら `grid-setup.ts` 1 ファイルの書き換えで
  個別登録に絞り込める

## 何を捨てたか

- **AG Grid Enterprise の導入は今回見送った。** 列固定・行グループ化・Excel 風のセル範囲選択 /
  コピペは MVP のスコープ外（`requests/00029_frontend_data_grid.md` の「やらないこと」）。
  ライセンス費用が発生するため、実際に要件化してから導入する
- **PrimeVue DataTable・Tabulator・ヘッドレス（TanStack Table）は採らなかった。** いずれも
  無効な選択肢ではなく、特にヘッドレス案は「既存フォーム層を流用できる」という別の軸で優れている。
  AG Grid で拡張の壁にぶつかったら再検討の余地がある
- **モジュールの個別登録・レガシー CSS テーマは今回使わない。** バンドルサイズより
  「参照を 1 か所に閉じる」ことと登録漏れの防止を優先した
- **バンドルサイズ対策（`defineAsyncComponent` による動的 import）は今回入れない。** 初期表示の
  実測をしていない段階で非同期化すると、ローディング表示の設計が余分に必要になる

## 覆すとしたら

- **列固定・行グループ化・Excel 風のセル範囲選択 / コピペが実際に要件化したとき。** AG Grid
  Enterprise のライセンスを購入し、`components/grid/grid-setup.ts` に
  `AllEnterpriseModule` の登録と `LicenseManager.setLicenseKey` を足すだけで切り替えられる
  （決定 2 の構成が前提）
- **バンドルサイズの増加が初期表示の実測で問題になったとき。** `components/grid/DataGrid.vue` を
  `defineAsyncComponent` で包む（呼び出し側は変更不要）か、モジュール登録を個別登録に絞る
- **AG Grid 自体の運用コスト（Community/Enterprise の境界を都度確認する手間）が見合わないと
  分かったとき。** その場合はヘッドレス案（TanStack Table + 既存フォーム層）への移行を検討する。
  ただしその時点までに作った `components/grid/` 配下のセル編集・検証の実装は作り直しになる
