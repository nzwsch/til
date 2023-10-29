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

### Rails Console(IRB)の色を表示しない

```ruby title="~/.irbrc"
IRB.conf[:USE_COLORIZE] = false
```

これはわざわざ指定しなくても困らない気がする。

体感上少し早くなる気がする。

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
