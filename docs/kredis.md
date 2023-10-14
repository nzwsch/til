# Kredis

Web Workerにも書いているが、Sidekiqのジョブに進捗を表示したいと思っている。
例えばNode.jsのBullというライブラリではあらかじめプログレスの表示機能が利用できる。
良くも悪くもSidekiqにはそこまで親切な機能はないため、自分で足りない機能を実装しなければならない。

一度だけやったことはあるのでできなくもないのだが、RailsでRedisのクライアントを直接呼び出すのは少々無骨すぎる。

そこで[少し調べると](https://gorails.com/forum/use-redis-as-a-way-to-update-a-model-along-a-process)、Railsがリリースしているkredisというライブラリがあるようだ。
今回はこのライブラリについてまとめていく。

余談だがGo Railsのビデオを見る限りでは*K-Redis(ケーレディス)*ではなく*Kredis(クレディス)*と発音していた。

### 基本

まずはkredisをインストールする。
この投稿時点では以下のバージョンで進める。

```text title=""
kredis (1.5.0)
```

READMEにも記載があるが、`./bin/rails kredis:install`コマンドを実行して生成されたYAMLファイルを編集する。

```yaml hl_lines="17"
# config/redis/shared.yml
production: &production
  url: <%= ENV.fetch("REDIS_URL", "redis://127.0.0.1:6379/0") %>
  timeout: 1

development: &development
  url: <%= ENV.fetch("REDIS_URL", "redis://127.0.0.1:6379/0") %>
  timeout: 1

  # You can also specify host, port, and db instead of url
  # host: <%= ENV.fetch("REDIS_SHARED_HOST", "127.0.0.1") %>
  # port: <%= ENV.fetch("REDIS_SHARED_PORT", "6379") %>
  # db: <%= ENV.fetch("REDIS_SHARED_DB", "11") %>

test:
  <<: *development
  db: <%= ENV.fetch("REDIS_SHARED_DB", "1") %>
```

デフォルトで生成されるファイルはdevelopment環境とtest環境で同じDBを参照しているのでテストコマンドの結果が変わってしまうのを防ぐために`db`を指定する。

### モデルの定義

```ruby
class Item < ApplicationRecord
  kredis_integer :progress
end
```

SQLのようにmigrationファイルを用意する必要はないので、モデルに直接`kredis_`+型を指定する。
最初は`kredis_counter`を指定していたが、`increment`あるいは`set`しかサポートしていなかったので`kredis_integer`にした。

またモデル経由で使う場合はすでにDBにコミットされている状態でなければならない。
初期化しただけのモデルで`progress`を呼び出そうとするとエラーが発生する。

```text title="Rails Console"
item = Item.create
=> #<Item:0x00007f19451bb880 id: 1, progress: nil, created_at: Sat, 14 Oct 2023 19:33:45.245662027 JST +09:00, updated_at: Sat, 14 Oct 2023 19:33:45.245662027 JST +09:00>
item.progress.value
  Kredis  (1.1ms)  Connected to shared
  Kredis Proxy (0.9ms)  GET items:1:progress
=> nil
```

注意点としては`kredis_integer`の初期値は`nil`である。
現在GitHubで公開されているmainブランチでは`kredis_integer :progress, default: 0`をサポートしている。
kredisのリリースが待ち遠しいが、`to_i`でも回避できるのでそこまで致命的なものでもない。

```text title="Rails Console"
item.progress.value = 1
  Kredis Proxy (0.4ms)  SET items:1:progress [1]
=> 1
item.progress.value
  Kredis Proxy (0.3ms)  GET items:1:progress
=> 1
```

`value=`あるいは`value`で値を参照できる。

```text title="Rails Console"
item.progress.key
=> "items:1:progress"
```

`key`は実際にRedisで使われているキーを取得できる。

```text title="Rails Console"
redis = Redis.new(host: 'localhost', port: 6379)
=> #<Redis client v5.0.7 for redis://localhost:6379/0>
redis.keys('*').include?(item.progress.key)
=> true
```

見たところSidekiqのキーと衝突はしないと思うが、ひょっとすると`kredis/items:1:progress`とかに書き換えたほうが安全かもしれない。

### SQLiteとの比較

もともとはモデルに対して`integer`のカラムを追加して、都度`update`していく実装だった。
データベースに値をコミットしていくのも悪くはないのだが、単純に`UPDATE`を実行するだけでもトランザクションが実行されるから安全である反面やっぱり時間がかかっている感じは否めない。

今回はbenchmark-ipsというgemを利用する。

`Item`クラスに`progress`というカラムを用意して、もう片方は`redis_progress`を定義しておき、それぞれ数値をセットするときのパフォーマンスを比較する。

```ruby
require "benchmark/ips"

ActiveRecord::Base.connection.execute <<-SQL
  CREATE TABLE IF NOT EXISTS items (
      id INTEGER PRIMARY KEY,
      progress INTEGER,
      created_at DATETIME,
      updated_at DATETIME
  );
SQL

class Item < ApplicationRecord
  kredis_integer :redis_progress
end

n = 100
a = Item.create
b = Item.create
Benchmark.ips(2) do |x|
  x.report("sqlite") do
    n.times do |i|
      a.update!(progress: i)
    end
  end

  x.report("kredis") do
    n.times do |i|
      b.redis_progress.value = i
    end
  end

  x.compare!
end

ActiveRecord::Base.connection.execute("DROP TABLE items;")
```

SQLite3をセットアップしたRailsの`rails runner`コマンドを実行した:

```text title=""
Warming up --------------------------------------
              sqlite     1.000  i/100ms
              kredis     5.000  i/100ms
Calculating -------------------------------------
              sqlite      2.050  (± 0.0%) i/s -      5.000  in   2.462847s
              kredis     64.403  (±10.9%) i/s -    130.000  in   2.051623s

Comparison:
              kredis:       64.4 i/s
              sqlite:        2.1 i/s - 31.41x  slower
```

体感上でも差はあったが、数値にしてみると想像以上に差があった。

ここまで差があるとやはりSQLiteよりもRedisでプログレスを管理する方がよさそうな気がする。

### Redisとの比較

では直接Redisクライアントと比較するとどうだろうか。

どちらもテーブル名は同じだが、`RedisItem`クラスはクラス変数にRedisのインスタンスを用意しておき、`get`あるいは`set`で値の更新と取得を繰り返す。

```ruby
require "benchmark/ips"

ActiveRecord::Base.connection.execute <<-SQL
  CREATE TABLE IF NOT EXISTS items (
      id INTEGER PRIMARY KEY,
      created_at DATETIME,
      updated_at DATETIME
  );
SQL

class RedisItem < ApplicationRecord
  self.table_name = "items"

  @@redis = Redis.new(host: "localhost", port: 6379)

  def progress
    @@redis.get("redis_item:#{id}:progress")
  end

  def progress=(value)
    @@redis.set("redis_item:#{id}:progress", value)
  end
end

class KredisItem < ApplicationRecord
  self.table_name = "items"

  kredis_integer :progress
end

n = 100
r = RedisItem.create
k = KredisItem.create
Benchmark.ips(2) do |x|
  x.report("redis") do
    n.times do |i|
      r.progress = i
      r.progress.to_i
    end
  end

  x.report("kredis") do
    n.times do |i|
      k.progress.value = i
      k.progress.value.to_i
    end
  end

  x.compare!
end

ActiveRecord::Base.connection.execute("DROP TABLE items;")
```

早速実行してみよう:

```text title=""
Warming up --------------------------------------
               redis     5.000  i/100ms
              kredis     2.000  i/100ms
Calculating -------------------------------------
               redis     50.752  (±17.7%) i/s -    100.000  in   2.030080s
              kredis     27.670  (±14.5%) i/s -     54.000  in   2.001809s

Comparison:
               redis:       50.8 i/s
              kredis:       27.7 i/s - 1.83x  slower
```

およそ2倍近くパフォーマンスに差が出るようだった。
この値を大きいと捉えるか、あるいは小さいと捉えるかの違いだ。

そこまで複雑でもないので自分で実装しても良い気もする。
しかしSQLiteの差ほど大きくは感じないし、極力Redisクライアントを意識したくはないので私はライブラリを採用する方を選んだ。
