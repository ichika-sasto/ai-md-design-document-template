---
status: closed
issue_date: 2026-09-09
start_date:
due_date:
end_date:
estimated_hours:
actual_hours:
version:
priority:
assignee:
parent:
labels:
  - "adr"
created_at: "2026-09-09 19:04:39"
updated_at: "2026-09-09 20:32:12"
---

# フロントエンドのフォーム実装方式

- 決定日: 2026-09-09
- 状態: 承認

## 背景

要望「vee-validate + shadcn-vue でフォームを部品化する」で、フロントエンドのフォームを
`vee-validate` + shadcn-vue の Form 系プリミティブに載せ替え、再利用可能な部品
（`components/form/TextField.vue` など）として切り出すことになった。

この移行にあたって、後から「なぜこうなっているか」を再調査させないために残しておくべき
決定が 3 つある。いずれも `SampleCreateForm` 1 件の実装では表面化しにくいが、
今後フォームが増えるたびに踏襲されることになる、フロントエンドの土台の決めごとである。

- zod のスキーマを vee-validate の検証に渡す**変換方法**
- 独自に作る再利用可能な部品コンポーネントの**置き場所**
- フォームの検証ルールを外部へ**公開する形**（関数か、スキーマそのものか）

## 選択肢

### 決定 1: zod → vee-validate の変換方法

vee-validate は `validationSchema` に `TypedSchema`（`parse` を持つオブジェクト）を渡す形で
スキーマベースの検証ライブラリと連携する。zod をそのまま渡すことはできない。

#### 案 A: 自前の `toTypedSchema` を実装する

`frontend/src/lib/vee-validate-zod.ts` に、zod の `safeParseAsync` を呼んで
`vee-validate` の `TypedSchema` 形式（`{ __type: 'VVTypedSchema', parse }`）に変換する
関数を約 30 行で実装する。

利点 /
- 依存パッケージが増えない
- zod 4 の挙動をそのまま使える（内部 API に依存しないので、zod のマイナーバージョンが
  上がっても壊れにくい）
- 実装しているのは `TypedSchema` の必須メンバー `parse` だけで、任意メンバーの
  `cast` / `describe` はこのリポジトリのどの部品も使わないため省ける

欠点 /
- 自前実装なので、vee-validate 本体の細かい仕様変更（`TypedSchema` インターフェースの変更）
  に追随する責任がこちらに残る

#### 案 B: 公式アダプタ `@vee-validate/zod` を `--legacy-peer-deps` で入れる

利点 /
- 実装コードを持たなくて済む
- vee-validate 本家がメンテナンスする

欠点 /
- **`@vee-validate/zod@4.15.1` の `peerDependencies` は `zod: ^3.24.0`。** 本リポジトリは
  `zod@4.5.4` を使っており、素の `npm install` は ERESOLVE で失敗する。
  `frontend/README.md` のセットアップ手順（`npm install --package-lock-only`）と、
  Dockerfile / `docker-entrypoint.sh` の `npm ci` の両方に `--legacy-peer-deps` を
  **恒久的に**持ち込むことになる
- 中身が zod 3 系の内部 API（`_def.typeName`、`ZodDefault._def.defaultValue()` など）に
  依存しており、zod 4 では静かに劣化する（`describe()` が常に
  `{ required: false, exists: false }` を返す等）。今回の実装では使わない機能だが、
  将来誰かが使ってハマる余地を残す

#### 案 C: typed schema を使わず、素の関数バリデータのまま vee-validate に渡す

利点 /
- 依存の衝突を一切気にしなくてよい

欠点 /
- **要望の MVP に反する。** 「`frontend-schema-layer` で扱っている Orval 生成 zod スキーマを
  `toTypedSchema` 経由でそのまま使う」ことが要望の必須要件であり、zod スキーマを
  そのまま検証に使えなくなる

### 決定 2: 独自ラップ部品の置き場所

shadcn-vue の Form 系プリミティブ（`Form` / `FormField` / `FormItem` / `FormControl` /
`FormMessage`）をラップした独自の再利用可能な部品（`TextField.vue` など）をどこに置くか。

#### 案 A: `components/form/`（新規フォルダ）に置く

利点 /
- shadcn-vue の `npm run ui:add` はレジストリからファイルをコピーしてくるコマンドで、
  対象は `components/ui/` 配下。ここへ独自ロジックを足すと、将来 `ui:add` を再実行した
  ときに上書きされる事故につながる
- 既存の `components/` は `ui/`（shadcn 生成物）・`layout/`（共通レイアウト）・
  `sample/`（機能別）という役割で 1 段掘る構成になっており、`form/` を足すのは
  この流儀に沿う
- 部品は機能に依存しない（再利用が目的）ので、`sample/` のような機能別フォルダにも属さない

欠点 /
- フォルダが 1 つ増える。`components/` 配下の見通しを保つ責任が増える

#### 案 B: `components/ui/` に直接置く

利点 /
- shadcn-vue の Form 系プリミティブと同じ場所にまとまる

欠点 /
- **`ui:add` の再実行で上書きされ得る。** shadcn-vue の CLI は対象コンポーネントの
  ファイルをレジストリの内容で丸ごと置き換えるため、同じディレクトリに独自ファイルを
  混ぜると、将来 `ui:add` で別のコンポーネントを追加したときに気づかず巻き込まれる
  リスクがある

## 決定 3: 検証ルールを公開する形

`frontend/src/lib/sample-schema.ts` は現状、`validateName(name): string | null` のような
**手書きの検証関数**を公開しており、コンポーネント側が送信時にこれを呼んでいる。

#### 案 A: vee-validate の typed schema を export する形に変える

`validateXxx` 関数を廃止し、`sampleCreateFormSchema` のような `toTypedSchema` 済みの
スキーマを `lib/sample-schema.ts` から export する。

利点 /
- `useForm({ validationSchema: sampleCreateFormSchema })` に直接渡せる。
  vee-validate が入力のたびの再検証・エラー表示位置の解決までを引き受けるため、
  コンポーネント側は検証ロジックを持たなくなる
- `frontend-schema-layer` の「制約値を手書きしない」規約はそのまま維持できる
  （制約値の取り込み元は変えない）

欠点 /
- 既存の `validateName` / `validateLabels` を直接呼んでいたテストやコードがあれば
  書き換えが要る（本リポジトリではテストランナー未導入・呼び出し元は対象の 3 フォームのみ
  のため影響は限定的）

#### 案 B: 手書きの `validateXxx` 関数を残し、`toTypedSchema` でラップするアダプタ関数を別途足す

利点 /
- 既存の `validateXxx` を残せるので、他の呼び出し元があっても壊れない

欠点 /
- 検証ロジックを「関数」と「スキーマ」の 2 つの形で持つことになり、
  どちらが正か分かりにくくなる。要望が目指す「部品を並べればバリデーション込みで動く」
  体験からも遠のく（結局コンポーネント側で `validateXxx` を呼ぶ経路が残る）

## 決定

**採るのは、決定 1 = 案 A（自前アダプタ）、決定 2 = 案 A（`components/form/`）、
決定 3 = 案 A（typed schema を export）。**

理由:

- **決定 1**: 公式アダプタの peer 依存衝突は今のバージョンの組み合わせでは解消できず、
  `--legacy-peer-deps` をセットアップ手順に恒久的に持ち込むコストのほうが、
  30 行程度の自前実装を持つコストより大きいと判断した。将来 `@vee-validate/zod` が
  zod 4 に対応したら、`lib/vee-validate-zod.ts` を消して import 元を差し替えるだけで
  案 B に切り替えられる
- **決定 2**: `components/ui/` は shadcn-vue の生成物置き場として役割を固定し、
  独自ロジックを混ぜないほうが、今後の `ui:add` 実行を安全にする
- **決定 3**: 要望が目指す「Vuetify のように部品を並べればバリデーション込みで動く」体験には、
  検証ロジックがスキーマとして部品に直接渡せる形が要る。関数呼び出しを残すと
  結局コンポーネント側に手続き的なコードが残ってしまう

## 何を捨てたか

- **`@vee-validate/zod` を将来まで含めて完全に排除したわけではない。** 今回は使わない
  という判断で、zod 4 対応版が出たら乗り換える余地を残している
- **`validateName` / `validateLabels` のような、フレームワーク非依存の検証関数は無くなる。**
  vee-validate の外（API 送信直前のチェックなど）でこれらを再利用したい場面が出たら、
  typed schema から `schema.parse()` を呼ぶ形に書き直す必要がある
- **`components/ui/` にコンポーネントを追加するときの手軽さは多少落ちる。** ラップ部品が
  必要かどうかを都度判断し、`components/form/` と `components/ui/` のどちらに置くか
  決める手間が増える

## 覆すとしたら

- **`@vee-validate/zod` が zod 4 系に対応したとき。** `lib/vee-validate-zod.ts` を
  廃止し、公式アダプタに乗り換える
- **`components/ui/` の生成物を編集しない運用が徹底できないと分かったとき**
  （例えば `ui:add` を使わず毎回手動コピーする運用に変わり、上書きの事故が起きなくなったとき）。
  そのときは `components/form/` と `components/ui/` を分ける理由が薄れる
- **フォームの検証を vee-validate 以外（例: 別のフォームライブラリ）に置き換えることになったとき。**
  決定 3 の「typed schema を export する」形は vee-validate 前提なので、
  ライブラリを変えるならこの決定ごと見直しになる
