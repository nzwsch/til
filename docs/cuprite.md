# Cuprite

RailsのシステムテストでChromeを操作させるドライバのFerrumを使いやすくしたのがCuprite。
イメージとしてはNode.jsのPuppeteerに近くて、CDPプロトコルというものを利用しているのでSeleniumに比べて挙動が軽いのだとか。

### インストールについて

実行環境に`chrome`のバイナリが必要。
Google Chromeをインストールしたくはなったので、Chromiumで代用する。
Ubuntu DesktopであればSeleniumはSnap経由でダウンロードできる。
Google Chromeは別途debパッケージをダウンロードする必要がある。

```text title=""
$ sudo snap install chromium
```

```ruby title="Gemfile"
group :development, :test do
  gem "capybara", "~> 3.39"
  gem "cuprite", "~> 0.14.3"
  gem "rspec-rails", "~> 6.0"
end
```

今回使うgemはこれ。

Systemテストのgemはわけもわからずたくさん入れてしまいがちだが、`cuprite`と`capybara`さえあれば動かせる。

### TL;DR

```ruby title="spec/support/cuprite.rb"
Capybara.javascript_driver = :cuprite

# スクリーンショットの保存先を指定できる
Capybara.save_path = Rails.root.join("tmp/capybara/")

# ChromeのコンテナからRailsサーバーにアクセスするための設定
if ENV["CHROME_URL"].present?
  ip_address = Socket.ip_address_list.find(&:ipv4_private?).ip_address

  # app host
  Capybara.app_host = "http://#{ip_address}"

  # server host
  Capybara.server_host = ip_address
end

# rspec settings
RSpec.configure do |config|
  # 前回のスクリーンショットファイルは残さない
  config.before(:all, type: :system) do
    FileUtils.rm_rf Capybara.save_path
  end

  config.before(:each, type: :system) do
    if ENV["CHROME_URL"].present?
      # timeoutを設定するのが重要。GitLab CIがほぼ必ず落ちるのを回避できる。
      options = {
        browser_options: { 'no-sandbox': nil },
        timeout: 20,
        url: ENV["CHROME_URL"],
      }
      driven_by(:cuprite, screen_size: [800, 600], options: options)
    elsif ENV["BROWSER"].present?
      driven_by(:cuprite, screen_size: [800, 600])
    else
      driven_by(:rack_test)
    end
  end
end
```

[Evil Martianの記事](https://evilmartians.com/chronicles/system-of-a-test-setting-up-end-to-end-rails-testing)も参考にしながらサポートスクリプトを書いた。
とりあえずこれを入れれば動く。
よくある`type: :system, js: true`みたいな書き方はしていない。
極力同じテストでJavaScriptありとなしの状態をテストできるようにしている。
ただし長い目で見ると破綻するかもしれないのでそこは注意が必要。

```text title=""
# rack_testのみのテスト
$ rspec

# システムのChromeとcupriteでテスト
$ BROWSER=1 rspec

# DockerとGitLab CIでテスト
$ docker-compose run --rm web rspec
```

ローカルで使うのはこの2種類。
基本的にはフットプリントの軽い`rack_test`前提でテストを書く。
CIで失敗したら`BROWSER=1`という環境変数でcupriteを有効にする。

```yaml title="docker-compose.yml"
version: "3"

services:
  web:
    build: .
    environment:
      - BROWSER=true
      - CHROME_URL=http://chrome:3000
      - RAILS_ENV=test
    volumes:
      - .:/web
    depends_on:
      - chrome

  chrome:
    image: browserless/chrome:latest
    container_name: chrome
```

Dockerでは少なくともRubyのコンテナにブラウザをインストールする必要はないので、別のコンテナを利用する。
Browserlessがよさそうだ。
純粋なChromeのコンテナではなさそうだけれども、自分でコンテナをビルドする手間を省ける。

```yaml title=".gitlab-ci.yml"
rspec:
  image: "$CI_REGISTRY_IMAGE"
  variables:
    RAILS_ENV: test
  script:
    - bundle exec rails db:prepare
    - bundle exec rspec

rspec system:
  image: "$CI_REGISTRY_IMAGE"
  variables:
    CHROME_URL: http://browserless-chrome:3000
    RAILS_ENV: test
  services:
    - browserless/chrome:latest
  script:
    - bundle exec rails db:prepare
    - bundle exec rspec spec/system
```

GitLab側のファイルはこれだけ。
注意すべきなのは`alias`を指定しないとホスト名が`browserless-chrome`になること。
あるいは`browserless__chrome`を指定しなければならない。
でもこれだけでSystemテストが動かせるのだからやっぱり便利だ。

### Capybara-Screenshot

[capybara-screenshot](https://github.com/mattheworiordan/capybara-screenshot)

Ferrum/Cupriteとは直接関係ないがあると便利。
スクリーンショットだけではなくHTMLも書き出してくれる。

`rack_test`のデバッグだと修正箇所がわかりやすい。

### 参考にしたURL

- [Railsでブラウザテストを「正しく」行う方法（翻訳）](https://techracho.bpsinc.jp/hachi8833/2023_06_28/95282)
    - 先程のEvil Martianの翻訳記事。
- [Rails System Tests In Docker](https://hint.io/blog/rails-system-test-docker)
- [How to implement Cuprite into Capybara](https://marsbased.com/blog/2022/04/18/cuprite-driver-capybara-sample-implementation/)
