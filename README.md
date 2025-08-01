# PostgreSQL セットアップガイド (Mac + Django)

## コンセプト

DjangoプロジェクトでPostgreSQLを使うための「ダウンロードから接続まで」の手順をまとめました。


---
## PostgreSQLのインストール（Homebrew）
任意のバージョンに合わせて `<version>` を変更してください。例：`14`, `15`, `16` など

```bash
brew install postgresql@<version>
```

インストールしたバージョンのサービスを起動
```bash
brew services start postgresql@<version>
```

状態確認コマンド（インストール済みのPostgreSQLサービスの確認）

```bash
brew services list
```
起動中なら　started

停止中なら　stopped


---

## PostgreSQLの初期化とログイン手順

### ログインする（すでに初期化済みの場合）

```bash
psql -U postgres
```
デフォルトでは postgres ユーザーでログインします。
ログインできない場合、初期化やサービス起動が必要なことがあります。

初期化が必要な場合（初回セットアップ時など）
```bash
initdb /usr/local/var/postgresql@<バージョン番号> -E utf8
```
initdb はデータベースクラスタを初期化するコマンドです（最初の一度だけでOK）。
<バージョン番号> には、インストールしたPostgreSQLのバージョンを指定します（例：14）。

例：
```bash
initdb /usr/local/var/postgresql@14 -E utf8
```


---

## 3. pg\_hba.confの設定

```bash
nano /usr/local/var/postgresql@<version>/pg_hba.conf
```

### trust から md5 に切り替え

```conf
local   all             all                                     md5
```

一時的に trust にする場合:

```conf
local   all             all                                     trust
```

---

## 4. PostgreSQL再起動

```bash
brew services restart postgresql@<version>
```

---

## 5. ユーザーとデータベース

### ユーザー作成

```sql
-- 管理者ユーザー（開発やデータベース操作用）
CREATE ROLE postgres WITH LOGIN SUPERUSER PASSWORD 'your_postgres_password';

-- アプリ用ユーザー（Djangoなどアプリからの接続専用）
CREATE ROLE app WITH LOGIN PASSWORD 'your_app_password';

-- 読み取り専用ユーザー（監査・分析ツールなどから使う）
CREATE ROLE readonly WITH LOGIN PASSWORD 'your_readonly_password';
```

### DB作成
PostgreSQLにログインした状態で、以下のSQLを実行すると新しいデータベースが作成される。

```sql
CREATE DATABASE your_db OWNER postgres;
```

### 権限付与

```sql
GRANT CONNECT ON DATABASE your_db TO app;
GRANT USAGE ON SCHEMA public TO app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app;
```

---

## 6. Django との接続


### settings.py のデータベース設定（PostgreSQL + 環境変数）
Django で PostgreSQL を使う場合、以下のように `settings.py` に記述します。

```python
from dotenv import load_dotenv
import os
load_dotenv()

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',  # PostgreSQL を使用
        'NAME': os.getenv('DB_NAME'),               # データベース名
        'USER': os.getenv('DB_USER'),               # ユーザー名
        'PASSWORD': os.getenv('DB_PASSWORD'),       # パスワード
        'HOST': os.getenv('DB_HOST'),               # ホスト（例: localhost）
        'PORT': os.getenv('DB_PORT'),               # ポート番号（例: 5432）
    }
}
```

### .env ファイルの内容例

```env
DB_NAME=your_db                  # 使用するデータベース名
DB_USER=postgres                 # データベース接続ユーザー名
DB_PASSWORD=yourpassword         # ← ここは「実際のパスワード」に置き換えてください（GitHubには絶対に公開しない）
DB_HOST=localhost                # 通常はローカル
DB_PORT=5432                     # PostgreSQLの標準ポート番号
```

---

## 7. 接続確認（データベースとDjangoの接続テスト）

以下のコマンドを実行して、Django と PostgreSQL の接続が正しくできているかを確認します。

```bash
python manage.py migrate
```
正常な出力の例：
実行すると、以下のように「Applying ... OK」という表示が複数出れば 接続成功 です。
```bash
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  ...
```
---

## 8. 補証

* `.env` は `.gitignore` に追加する
* `postgres` は開発用として利用
* 本番では `app` ユーザーのみを使用
* psql 内のコマンド:

  * `\du` = ユーザー一覧
  * `\l` = データベース一覧
  * `\dt` = テーブル一覧
 
---

## 9 .gitignore に追加しておくべき内容例
.env

__pycache__/

*.pyc

db.sqlite3


---

これでDjangoプロジェクトでPostgreSQLを安全に利用する基盤が整いました。

---

# PostgreSQL連携作業ログ

---

## 概要

Djangoアプリ `private_diary` を SQLite から PostgreSQL に切り替えるための構築作業を行った。  
環境変数を `.env` ファイルで管理し、セキュリティを保ちながら接続設定を完了。  
その後、マイグレーションを実行し、データベース連携が正常に動作することを確認した。

---
## 作業ステップ

### 1. PostgreSQLの起動確認

```bash
brew services list
```

### 2. psql にログインし、既存のロールとDBを削除

```sql
DROP DATABASE IF EXISTS your_db;
DROP ROLE IF EXISTS app;
DROP ROLE IF EXISTS your_user;
```

### 3. 新しいロールとデータベースを作成

```sql
CREATE ROLE app WITH LOGIN PASSWORD 'apppass';
CREATE DATABASE your_db OWNER app;
```

### 4. `.env` ファイルの設定

> `.env`（Gitに公開しない！）

```env
DB_NAME=your_db
DB_USER=app
DB_PASSWORD=apppass
DB_HOST=localhost
DB_PORT=5432
```

### 5. `settings.py` に環境変数から読み込む設定を追加

```python
import environ

env = environ.Env()
environ.Env.read_env()

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': env('DB_HOST'),
        'PORT': env('DB_PORT'),
    }
}
```

### 6. Djangoマイグレーション実行

```bash
python manage.py migrate
```

`migrate` 成功を確認。

---

## セキュリティ対策

- `.env` ファイルは `.gitignore` に追加済み
- パスワードは仮の開発用（本番環境ではより複雑なものに変更予定）

---
## PostgreSQLの現状確認チェックリスト
| チェック項目                    | 確認方法・コマンド例                           |                 |
| ------------------------- | ------------------------------------ | --------------- |
| ✅ PostgreSQLがインストールされているか | \`brew list                          | grep postgres\` |
| ✅ PostgreSQLのサービスが起動中か    | `brew services list` で `started` を確認 |                 |
| ✅ ローカルで接続できるか             | `psql -U postgres`                   |                 |
| ✅ 現在のデータベース一覧             | `\l`（psql内で実行）                       |                 |
| ✅ 現在のロール（ユーザー）一覧          | `\du`（psql内で実行）                      |                 |
| ✅ 使用中のデータベースへ接続できるか       | `\c データベース名`                         |                 |
| ✅ テーブル一覧                  | `\dt`（Djangoのマイグレーション済みであれば表示される）    |                 |


---


## 今後の予定（次のフェーズ）

- [ ] ユーザー認証（ログイン・登録）機能の追加
- [ ] 投稿ユーザーの絞り込み（ログインユーザーの投稿のみ表示）
- [ ] データベースへの初期投入（superuserの作成など）




