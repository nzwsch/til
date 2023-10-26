# Rclone

Railsなどでアップロードしたファイルを別のコンピュータへ移動したい場合は`docker cp`コマンドなどを利用していたのだけれども、ようやくRcloneというコマンドに落ち着いた。
これは複数のクラウドサービスに対応しており、Minioのクライアントとしても使えるようだ。

### インストール

```sh title=""
$ sudo apt install rclone
```

### `rclone config`

```text title=""
$ rclone config
n/s/q> n
name> minio
Storage> 4
provider> 7
env_auth> 1
access_key_id> USWUXHGYZQYFYFFIT3RE
secret_access_key> MOJRH0mkL1IPauahWITSVvyDrQbEEIwljvmxdq03
region> 1
endpoint> http://192.168.1.106:9000
server_side_encryption> 1
sse_kms_key_id> 1
Edit advanced config? (y/n)
y) Yes
n) No (default)
y/n> n
```

出力をだいぶ省略しているが、対話的な処理で入力していけばよい。
Storageを4にしてproviderが7になっていればあとはデフォルトで適当に進めていくだけだ。

`rclone config show`コマンドで設定ファイルの内容を確認できるようだ。

```text title=""
$ rclone config show
[minio]
type = s3
provider = Minio
env_auth = false
access_key_id = USWUXHGYZQYFYFFIT3RE
secret_access_key = MOJRH0mkL1IPauahWITSVvyDrQbEEIwljvmxdq03
endpoint = http://192.168.1.106:9000
bucket_acl = private
```

### Minio側のACL

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::satonocrown"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:*"
      ],
      "Resource": [
        "arn:aws:s3:::satonocrown/*"
      ]
    }
  ]
}
```

今回は`satonocrown`というbucketを作成した。
自分のbucketの範囲内でなら基本的に何でも許可するといったイメージだ。

### `rclone listremotes`

```text title=""
$ rclone listremotes
minio:
```

アクセスできるディレクトリの一覧を取得できる。

### `rclone listremotes`

```text title=""
$ rclone listremotes
minio:
```

### `rclone lsd`

アクセスできるディレクトリの一覧を取得できる。

```text title=""
$ rclone lsd minio:
          -1 2023-10-26 17:48:13        -1 satonocrown
```

他にも`ls`、`lsf`などが使えるようだ。
`lsd`はディレクトリのみで`lsf`はファイルのみ、`ls`はどちらも取得できる。
さきほどのACLもここで確認できる。

### `rclone copy`

ファイルをアップロードできる。

```text title=""
$ rclone copy ./satonocrown/public/uploads/ minio:satonocrown/uploads
```

Minioのブラウザ経由で確認するとRailsのアップロードディレクトリがコピーできていた。
