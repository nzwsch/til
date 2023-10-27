# Ubuntu Desktop

Ubuntu全般について

### ImageMagickを消す

```
$ sudo rm /usr/share/applications/display-im6.q16.desktop
```

謎の白いアイコンが消える。
実行してみてもよくわからない画像が出てくるだけ。
imagemagickをインストールするともれなくついてくるが消して問題なさそう。

### nip2を消す

```
$ sudo apt purge nip2
```

こちらは`libvips-dev`をインストールすると一緒にインストールされるGUI版?のプログラム。
アイコンが微妙なのでプログラムごと消す。

### mpvを消す

```
$ sudo apt purge mpv
```

`yt-dlp`をインストールしたときにインストールされているようだった。
そもそも起動すらしない。
