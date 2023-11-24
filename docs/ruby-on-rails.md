# Ruby on Rails

Rails全般

### `field_with_errors`クラスを消す

フォームでvalidationエラーが出ると新しい`div`で囲われてしまう。

レイアウトが崩れるのをとりあえず防ぎたい:

```ruby
ActionView::Base.field_error_proc = Proc.new do |html_tag, instance|
  html_tag.html_safe
end
```

参考URL: [Remove Rails field_with_errors wrapper](https://coderwall.com/p/s-zwrg/remove-rails-field_with_errors-wrapper)

### Rails Console(IRB)の候補を表示しない

Ruby 3系だと、Rails Console(IRB)の入力中に候補が表示されるようになった。

```text title=""

                 APP_PATH             █
                 ARGF                 █
                 ARGV                 █
                 AbstractController   █
                 ActionCable          █
                 ActionController     █
                 ActionDispatch       █
                 ActionPack           █
                 ActionView           █
                 ActiveJob
                 Addrinfo
```

候補表示そのものは必要だと思うのだが、タイピングの途中に出す必要はないと思う。

それどころかIRBの表示もどこかバグっぽくて、この表示のせいで入力している文字が消えるという致命的なバグがある。
こういうのはPRYとかのgemでオプトインするべきであって、IRBは可能な限り単純でいてほしかった。
とはいえ、カスタマイズが可能なのでホームディレクトリに`.irbrc`を作成する。

```ruby title="~/.irbrc"
IRB.conf[:USE_AUTOCOMPLETE] = false
```

とてもよくなった。

一応環境変数でも変更できるようだ。

```bash title="~/.bashrc"
export IRB_USE_AUTOCOMPLETE=false
```

### Rails Console(IRB)のプロンプトをカスタマイズする方法

せっかく`.irbrc`を作ったので、古き良きIRBにする。

```ruby title="~/.irbrc"
# シンプルなプロンプトを使う
IRB.conf[:PROMPT][:MY_PROMPT] = {
  :PROMPT_I => ">> ",
  :PROMPT_S => "?> ",
  :PROMPT_C => "-> ",
  :RETURN => "=> %s\n"
}
IRB.conf[:PROMPT_MODE] = :MY_PROMPT
```

この表示がデフォルトだといいのに。

```text title=""
$ rails c
Loading development environment (Rails 7.0.8)
>>
```

非常によくなった。

### `.railsrc`

とても今更ではあるのだが、毎回`rails new`するときに必ず`-T`などのオプションを指定している。
MiniTestはまだよいのだが、`--skip-action-mailbox`などは大文字で省略できないので大抵は`rails new --minimal`か`rails new --help`で一覧からいちいち選択していた。

あまり気に留めていなかったのだが、`.irbrc`がホームディレクトリにあることを考えると`.railsrc`も同様にあってもよいのではないかと思えてきた。

```text title="~/.railsrc"
--database=postgresql
--skip-keeps
--skip-action-mailer
--skip-action-mailbox
--skip-action-text
--skip-active-storage
--skip-jbuilder
--skip-test
```

とりあえずほぼ毎回選ぶであろう選択肢を選んだ。

以下は選択したコメント:

`--database=postgresql`

ここ最近はPostgresではなくSQLiteで書き始めることが多い。
`db:system:change`があるとはいえ、デプロイするには必ずPostgresに変更しなければならないので指定しておく。
単純に毎回`-d=postgresql`をタイプするのが面倒だっただけかもしれない。

`--skip-keeps`

これは逆に今回から採用することに決めた。
`tmp`とか`log`をGitLabに表示しておきたいから残していたのでこれは`--no-skip-keeps`にするかもしれない。

`--skip-action-mailer`

ActionMailerはいつかは使いたいと思っているものの、だいたいは別の手段に落ち着く。

`--skip-action-mailbox`

正直興味はあるものの、ActionMailer使わないのであれば採用する理由はない。

`--skip-action-text`

Trixはリリースされた当時は非常にニーズを捉えていると思っていた。
ただし毎回ブログを作りたいわけではないし、もしブログを作るとしてもMarkdownをサポートしたいと思う。
そうなるとTrixを使いたいと思う機会はなかなか想定できない。
もう少しクセのないエディタならまた違った捉え方だったかもしれない。

`--skip-active-storage`

ActiveStorageは一時期使おうとしていたのだが、やはりCarrierWaveのほうが私は好きだ。
これも将来的にこの中に含まれなくなる可能性はあるかもしれないが、`rails routes`が汚染されるのがどうも気に入らない。

`--skip-jbuilder`

JBuilderはMVCを体現していると思うし、あまり嫌いではない。
でもJSONはシリアライズするほうがより一般的だと思う。

`--skip-test`

ミニマリストという感じがしてはかっこいい。
でもそれだけである。
MiniTestでもテスト書けなくはないけれど、やはりRSpecに比べるとわざわざこれを好んで書く理由はない。

`--no-skip-action-cable`

この設定はリストに含まれていないけれども、ほぼ必ずといっていいほどActionCableは除外していた身からするとこれは大きな進歩だと思う。
利用するにはRedisが必須だけども、Rails自体はRedisを全く設定しなくても動く。
WebSocketというのも敬遠していた理由だけれども、今後Hotwireを使っていきたいのが大きい。

`--template=TEMPLATE`

正直試してみたかったのだが、Gemfileはできる限り`bundle add`で悲観的に追加したい。
そう考えると他にできそうなことは少ないのでスルーした。

### .rubocop.ymlをサーバー経由する

たまたまKredisのRuboCopを見ていたら記述がたったのこれだけであった:

```yaml title=".rubocop.yml"
inherit_from: https://raw.githubusercontent.com/rails/rails/main/.rubocop.yml
```

これをやることによって手元の編集をいちいちする必要がないというか、複数のプロジェクト間で毎回似たようなファイルを定義する必要がなくなった。

問題はこれをどこで管理するかなのかだが、GitHubでオープンソース管理しているのであればGitHubに任せるのが理にかなってはいる。
しかし私の場合は極力GitHubよりもGitLabでリポジトリを管理したいので、GitLabのSnippet機能を使うことにした。

このアプローチで少々厄介なのは`rubocop`コマンドを実行するとローカルに少々長いファイルが生成されてしまうようだ:

```text title=""
.rubocop-https---raw-githubusercontent-com-rails-rails-main--rubocop-yml
```

そのためVSCodeで以下のような設定を追加した。

```yaml title="settings.json"
{
  "files.exclude": {
    "**/.rubocop-*": true
  }
}
```

これは結構強力な設定で、Explorerからも除外する設定だ。
これもしばらく運用してみてローカル管理がしっくりくるのか、いずれ結論を出したいと思う。

### SVGを表示する方法

あまり見慣れない`raw`というヘルパーメソッドを使用している。

```ruby title="icon_helper.rb"
module IconHelper
  def svg(file_name)
    icons_path = Rails.root.join("app/assets/icons").join(file_name)
    File.open(icons_path) do |f|
      raw f.read
    end
  end
end
```

参考URL: [How do I display SVG image in Rails?](https://stackoverflow.com/questions/36986925/how-do-i-display-svg-image-in-rails)

### テンプレートのログを隠す

最近ではView Componentを使えば似たような事ができなくもない事に気がついたのだけれども、モデルの数だけ`render`が呼ばれるのが鬱陶しく感じる。

とはいえSQLのログやコントローラのパラメータは表示しておきたいので次のオプションを有効にすることにした。`rails dev:cache`みたいにファイルで制御できるようにしてもよさそうではある。

```ruby title="config/initializers/action_view.rb"
Rails.application.config.action_view.logger = nil
```

参考URL: [Hide rendering of partials from rails logs](https://stackoverflow.com/questions/17312076/hide-rendering-of-partials-from-rails-logs)