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
  - "usecase"
created_at: "2026-09-14 09:48:01"
updated_at: "2026-09-14 15:39:48"
---

# マスターユーザーを起動時に同期する

## このユースケースは何をするか

アプリ起動時に、環境変数で持つマスターユーザーのログインID・メールアドレス・パスワードで
`User` / `LocalAuth` を upsert する。存在しなければ作り、既にあればパスワードを環境変数の値に
合わせる。特別扱いの認証経路は作らず、通常のユーザーとして扱う。

要望 00027 で実装済み（`app/core/startup.py` の `sync_master_user()`、
`app/application/auth/usecase.py` の `SyncMasterUserUseCase`）。API エンドポイントではなく、
アプリ起動処理の一部（バッチ的な処理）として実行される。

## 入力

| 論理名 | 物理名 | 型 | 必須 | 備考 |
| --- | --- | --- | --- | --- |
| マスターユーザーのログインID | `MASTER_USER_LOGIN_ID` | 文字列 | ○ | 環境変数から読む。③で確定 |
| マスターユーザーのメールアドレス | `MASTER_USER_EMAIL` | 文字列 | ○ | 環境変数から読む。③で確定 |
| マスターユーザーのパスワード | `MASTER_USER_PASSWORD` | 文字列 | ○ | 環境変数から読む。平文はログに出さない。③で確定 |

## 出力

| 論理名 | 物理名 | 型 | 備考 |
| --- | --- | --- | --- |
| upsert された User | `UserResult` | オブジェクト | 戻り値を使う呼び出し元は無い（起動シーケンスの一部）。③で確定 |

**副作用**

- `login_id`（`MASTER_USER_LOGIN_ID`）で `User` を検索する（③で確定：**upsert の判定キーは
  `login_id`**）。無ければ `next_user_code_number()` でユーザーコードを採番して `users` に
  1 行 INSERT、同時に `local_auths` に 1 行 INSERT する。
- 既にあれば、**`email` も毎回 env の値に合わせて更新する**（③で確定。env が「正」なので
  ログインID以外は毎回追随させる）。`local_auths` のパスワードハッシュも env のパスワードから
  作り直して更新する。

## 使用するアグリゲート / エンティティ

| アグリゲート | 読み書き | 何のために |
| --- | --- | --- |
| [017_user](017_user.md) | 書き換える | マスターユーザーの `User` / `LocalAuth` を作る、またはパスワードを合わせる |

## 例外

| 条件 | 結果 |
| --- | --- |
| 環境変数が 1 つでも設定されていない | **`RuntimeError` で起動を止める**（既存の `get_database_url()` と同じ方針。文面もそれに倣う）。③で確定 |
| 同じログインIDだが `email` が既存の別ユーザーの `email` と衝突する | `ConflictError`（一意制約違反）。**起動時なのでスタックトレースをログに出し、起動は続行しない**（握らない）。③で確定 |
| 起動時に DB が無い / テーブル未作成（`SQLAlchemyError`） | **その例外だけは握って `logger.error` を出し、起動は続行する**（③で確定。マイグレーション未適用でも起動が落ちないようにするため。適用後は `docker compose restart api` で同期する） |

## 未決

無い。

③で確定し、ここから外れたもの: 環境変数の名前（`MASTER_USER_LOGIN_ID` / `MASTER_USER_EMAIL` /
`MASTER_USER_PASSWORD`）、upsert の判定キー（`login_id`）、環境変数が欠けているときの扱い
（起動を止める）、メールアドレスを毎回同期するか（毎回同期する）、DB エラー時の扱い
（`SQLAlchemyError` のみ握って起動続行）。ユーザーコードの自動生成規則は
[017_user](017_user.md) を参照（`USER` + 10 桁連番、`user_code_seq`）。
