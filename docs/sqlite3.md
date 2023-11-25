# SQLite3

DBは普段GUIでしか扱わないのだけれども、バックアップとリストアについて調べておく。

参考URL: [How To Use The SQLite Dump Command](https://www.sqlitetutorial.net/sqlite-dump/)

### インストール

```text title=""
sudo apt install sqlite3
```

### データベースのdump

```text title="sqlite3 ./db/development.sqlite3"
SQLite version 3.37.2 2022-01-06 13:25:41
Enter ".help" for usage hints.
sqlite> .output db/dump.sql
sqlite> .dump
sqlite> .exit
```

### 特定のテーブルをdump

```text title="sqlite3 ./db/development.sqlite3"
SQLite version 3.37.2 2022-01-06 13:25:41
Enter ".help" for usage hints.
sqlite> .output db/videos.sql
sqlite> .dump videos
sqlite> .exit
```

これひょっとすると対話型のコマンドで実行しなくてもいけるのではという気持ちもある。

### 特定のテーブルを`db:seed`でリストア

ここからが本題である。

先程のコマンドで`dump.sql`あるいは`videos.sql`が取得できた前提で実行する。
まずはテーブル全体を初期化する:

```text title=""
rails db:drop db:create db:migrate
Dropped database 'db/development.sqlite3'
Dropped database 'db/test.sqlite3'
Created database 'db/development.sqlite3'
Created database 'db/test.sqlite3'
== 20231123063711 CreateVideos: migrating =====================================
-- create_table(:videos)
   -> 0.0020s
== 20231123063711 CreateVideos: migrated (0.0027s) ============================
```

つづいてseeds.rbを次のように更新する:

```ruby title="db/seeds.rb"
videos_sql = File.read("db/videos.sql")
result = ActiveRecord::Base.connection.execute(videos_sql)
```

`rails db:seed`してみたところ残念ながらDBはリストアできていなかった。

試しに`INSERT INTO`行のみを実行してみたところ、レコードは1件のみ作成されていた。つまり複数行は実行できないようだ。

先程のseeds.rbを修正する:

```ruby title="db/seeds.rb" hl_lines="4"
videos_sql = File.read("db/videos.sql")
videos_sql.each_line do |sql|
  if sql.match?(/^INSERT INTO/)
    ActiveRecord::Base.connection.execute(sql)
  end
end
```

いい感じだ。
たまたま今回はリストア対象のテーブルが1つしかなかったのでこれで十分だ。

参考URL: [Execute SQL-Statement from File with ActiveRecord](https://stackoverflow.com/questions/31403426/execute-sql-statement-from-file-with-activerecord)