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

## 7. 接続確認

```bash
python manage.py migrate
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

これでDjangoプロジェクトでPostgreSQLを安全に利用する基盤が整いました。

