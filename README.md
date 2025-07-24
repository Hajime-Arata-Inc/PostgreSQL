# PostgreSQL セットアップガイド (Mac + Django)

## コンセプト

DjangoプロジェクトでPostgreSQLを使うための「ダウンロードから接続まで」の手順をまとめました。

---
PostgreSQLのインストール（Homebrew）
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
-- 管理者ユーザー（開発やデータベース操作用）
CREATE ROLE postgres WITH LOGIN SUPERUSER PASSWORD 'your_postgres_password';

-- アプリ用ユーザー（Djangoなどアプリからの接続専用）
CREATE ROLE app WITH LOGIN PASSWORD 'your_app_password';

-- 読み取り専用ユーザー（監査・分析ツールなどから使う）
CREATE ROLE readonly WITH LOGIN PASSWORD 'your_readonly_password';
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
DB_NAME=private_diary_db         # 使用するデータベース名
DB_USER=postgres                 # データベース接続ユーザー名
DB_PASSWORD=yourpassword         # ← ここは「実際のパスワード」に置き換えてください（GitHubには絶対に公開しない）
DB_HOST=localhost                # 通常はローカル
DB_PORT=5432                     # PostgreSQLの標準ポート番号
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

