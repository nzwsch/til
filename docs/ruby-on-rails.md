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
