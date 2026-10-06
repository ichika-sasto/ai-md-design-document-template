---
status: closed
issue_date: 2026-09-14
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
  - "aggregate"
created_at: "2026-09-14 09:48:01"
updated_at: "2026-09-14 15:39:48"
---

# User アグリゲート

## このアグリゲートは何か

利用者を表す `User`（ルート）と、ID / パスワードによる認証情報を表す `LocalAuth`（子エンティティ、
1 ユーザーにつき高々 1 件）の 2 テーブル構成。将来 OIDC 認証を足すときに `User` はそのまま、
認証手段のテーブル（例: `OidcAuth`）を増やすだけで済むようにするため、認証手段を `User` から分離してある。

要望 00027 で実装済み（`app/domain/user/`、`app/infrastructure/user/`、マイグレーション
`20260914_9f9982a91989_add_auth.py`）。

## 不変条件

- **ログインID・メールアドレス・ユーザーコードはそれぞれ全体で一意**。
- **`LocalAuth` は 1 ユーザーにつき高々 1 件**（1:1）。ログインID / パスワード認証を使わない
  ユーザーは今回想定していない（OIDC 単独ユーザーは対応外）。
- **パスワードは `LocalAuth` にハッシュ化して保存し、平文を持たない**（完了条件に明記）。
- **`User` は別アグリゲートのエンティティを持たない**（他アグリゲートへの参照が必要になっても
  ID だけを持つ。現状 `User` は他アグリゲートを参照しない）。

## エンティティ

| 論理名 | 物理名 | ルート | 備考 |
| --- | --- | --- | --- |
| ユーザー | `users` | ○ | ルート。`User` |
| ローカル認証情報 | `local_auths` |  | `LocalAuth`。単体では存在せず、必ず `User` に 1:1 で紐づく。**ドメインエンティティとしては `user_id` を保持しない**（③で決定。`SampleItem` と同じ方針で、親への参照列は ORM モデル側だけが持ち、取得時は `selectinload` で紐付ける） |

## 属性

### ユーザー

| 論理名 | 物理名 | 型 | 必須 | キー | 備考 |
| --- | --- | --- | --- | --- | --- |
| ID | `id` | UUID | ○ | PK | ドメイン側で採番する（`uuid4`）。既存ドメイン（`sample` / `category`）と同じ方針 |
| ログインID | `login_id` | 文字列(50) | ○ | 一意（`uq_users_login_id`） | 3〜50 文字、半角英数字と `.` `_` `-`（`^[A-Za-z0-9._-]+$`）。③で確定 |
| メールアドレス | `email` | 文字列(3〜254) | ○ | 一意（`uq_users_email`） | 簡易形式検証（`^[^@\s]+@[^@\s]+\.[^@\s]+$`）。③で確定 |
| ユーザーコード | `user_code` | 文字列(14) | ○ | 一意（`uq_users_user_code`） | `USER` + 0 埋め 10 桁の連番（`^USER\d{10}$`、例 `USER0000000001`）。採番は PostgreSQL のシーケンス `user_code_seq`。書式の組み立ては `domain/user/entity.py` の `build_user_code()` に一元化する。③で確定 |

### ローカル認証情報

| 論理名 | 物理名 | 型 | 必須 | キー | 備考 |
| --- | --- | --- | --- | --- | --- |
| ID | `id` | UUID | ○ | PK | ドメイン側で採番する（`uuid4`） |
| ユーザーID | `user_id` | UUID | ○ | 一意（`uq_local_auths_user_id`）, FK → ユーザー.ID（`fk_local_auths_user_id_users`、`ON DELETE CASCADE`） | 1:1。同一アグリゲート内なので外部キーを張る。**この列は ORM モデルのみが持ち、`LocalAuth` ドメインエンティティは `user_id` を属性として持たない**（③で決定） |
| パスワードハッシュ | `password_hash` | 文字列(255) | ○ |  | **Argon2id**（`argon2-cffi` の `PasswordHasher` を既定パラメータのまま使用。`passlib` は挟まない）でエンコードした文字列（`$argon2id$...`、実測 97 文字前後）。③で確定 |

## インデックス

無い。一意制約（下記）が索引を兼ねる。

## 複合ユニーク

複合（複数カラムの組み合わせ）ではないが、単一カラムの一意制約として以下を持つ。

| 名前 | 対象エンティティ | 対象カラム | 備考 |
| --- | --- | --- | --- |
| `uq_users_login_id` | ユーザー | `login_id` | ログインIDの重複禁止 |
| `uq_users_email` | ユーザー | `email` | メールアドレスの重複禁止 |
| `uq_users_user_code` | ユーザー | `user_code` | ユーザーコードの重複禁止 |
| `uq_local_auths_user_id` | ローカル認証情報 | `user_id` | 1 ユーザーにつき 1 件までの 1:1 制約 |

## 状態

無い。`User` / `LocalAuth` とも `status` に相当する属性を持たない
（有効化・無効化・アカウントロックは今回のスコープ外。要望の「やらないこと」参照）。

## イベント

無い。今回 MVP では `User` を作る経路が「マスターユーザーの起動時同期」だけで、
ユーザー登録 API・パスワード変更 API は見送っているため、`Sample` のような
`available_events()` / `EDITABLE_FIELDS` の仕組みは今回持たせない
（[023_sync_master_user](023_sync_master_user.md) が直接 `User` / `LocalAuth` を組み立てて upsert する）。

## 状態遷移図

状態もイベントも無いため、遷移図は無い。

## 未決

- **OIDC 認証を足すときの拡張の形が未定。** `OidcAuth` のような新しいテーブルを追加するだけで
  済むのか、`LocalAuth` 側にも手を入れる必要があるのかは、OIDC 接続自体が別チケットのため未検討。
- **ユーザーの有効化・無効化・削除の経路が無い。** 今回のスコープ外（要望の「やらないこと」参照）。
- **`login_id` と `user_code` を変える経路が無い。** `User` エンティティのメソッドは
  `change_email()` と `set_password_hash()` のみで、ログインIDとユーザーコードを変更する
  メソッドを持たない（⑤で確認。`app/domain/user/entity.py`）。値を変えるかどうかは未検討。

③で確定し、ここから外れたもの: ユーザーコードの生成規則、ログインID・メールアドレスの文字種・
最大長、パスワードハッシュのアルゴリズム（上の「属性」参照）。
