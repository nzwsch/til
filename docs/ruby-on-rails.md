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
