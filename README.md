# PostgreSQL セットアップガイド (Mac + Django)

## コンセプト

DjangoプロジェクトでPostgreSQLを使うための「ダウンロードから接続まで」の手順をまとめました。

---

## 1. PostgreSQLのインストール

```bash
brew install postgresql@14
brew services start postgresql@14
```

状態確認:

```bash
brew services list
```

---

## 2. 初期化とログイン

```bash
psql -U postgres
```

初期化が必要な場合:

```bash
initdb /usr/local/var/postgresql@14 -E utf8
```

---

## 3. pg\_hba.confの設定

```bash
nano /usr/local/var/postgresql@14/pg_hba.conf
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
brew services restart postgresql@14
```

---

## 5. ユーザーとデータベース

### ユーザー作成

```sql
CREATE ROLE postgres WITH LOGIN SUPERUSER PASSWORD 'yourpassword';
CREATE ROLE app WITH LOGIN PASSWORD 'apppass';
CREATE ROLE readonly WITH LOGIN PASSWORD 'readonlypass';
```

### DB作成

```sql
CREATE DATABASE private_diary_db OWNER postgres;
```

### 権限付与

```sql
GRANT CONNECT ON DATABASE private_diary_db TO app;
GRANT USAGE ON SCHEMA public TO app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app;
```

---

## 6. Django との接続

### .env 例

```env
DB_NAME=private_diary_db
DB_USER=postgres
DB_PASSWORD=yourpassword
DB_HOST=localhost
DB_PORT=5432
```

### settings.py の設定

```python
from dotenv import load_dotenv
import os
load_dotenv()

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME'),
        'USER': os.getenv('DB_USER'),
        'PASSWORD': os.getenv('DB_PASSWORD'),
        'HOST': os.getenv('DB_HOST'),
        'PORT': os.getenv('DB_PORT'),
    }
}
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

