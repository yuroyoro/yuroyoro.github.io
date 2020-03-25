---
title: Docker for MacでUID指定しつつssh-agentにaddしたホスト側の秘密鍵をコンテナ内で使いたい
tags: Docker SSH
author: yuroyoro
slide: false
---
コンテナ内で外部にホスト側の秘密鍵を使ってsshしたい場合があります。githubとか、開発用のサーバーとか色々。

そんな場合は、秘密鍵をボリュームマウントするという力技もありますが、 ホスト側の `ssh-agent` をコンテナ内でも使う方法がスマートです。

# コンテナ内でSSH agent forwardingする

1. ホストで `ssh-agent` を起動する
2. 秘密鍵を `ssh-add` しておく
3. 環境変数 `SSH_AUTH_SOCK` で指定されるunix domain socketをコンテナにボリュームマウントして `docker run`

```shell
# ssh-agent起動
$ eval $(ssh-agent -a /tmp/ssh-auth-sock)
Agent pid 17194

# 秘密鍵をadd
$ ssh-add ~/.ssh/id_rsa
Identity added: /home/ozaki/.ssh/id_rsa (/home/ozaki/.ssh/id_rsa)

# SSH_AUTH_SOCKをマウントし環境変数指定してssh
$ docker run -it -v ${SSH_AUTH_SOCK}:${SSH_AUTH_SOCK} -e SSH_AUTH_SOCK="${SSH_AUTH_SOCK}" --rm alpine:3.4 /bin/sh -c "apk -q update && apk -q add dropbear-ssh && ssh -T git@github.com"

Host 'github.com' is not in the trusted hosts file.
(ssh-rsa fingerprint md5 16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48)
Do you want to continue connecting? (y/n) y
Hi yuroyoro! You've successfully authenticated, but GitHub does not provide shell access.
```

Linuxの場合はこれでうまく行くのですが、MacOSXの場合は `SSH_AUTH_SOCK` をそのままマウントしてもうまくいきません。代わりに、 以下のドキュメントのように `/run/host-services/ssh-auth.sock` という謎のファイルをマウントしてあげる必要があります。

[File system sharing \(osxfs\) \| Docker Documentation](https://docs.docker.com/docker-for-mac/osxfs/#ssh-agent-forwarding)


```shell
# macだとうまくいかない
$ docker run -it -v ${SSH_AUTH_SOCK}:${SSH_AUTH_SOCK} -e SSH_AUTH_SOCK="${SSH_AUTH_SOCK}" --rm alpine:3.4 /bin/sh -c "apk -q update && apk -q add dropbear-ssh && ssh -T git@github.com"

Host 'github.com' is not in the trusted hosts file.
(ssh-rsa fingerprint md5 16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48)
Do you want to continue connecting? (y/n) y
ssh: Failed to connect to agent

ssh: Connection to git@github.com:22 exited: No auth methods could be used.

# /run/host-services/ssh-auth.sock をマウントするとうまくいく
$ docker run --mount type=bind,src=/run/host-services/ssh-auth.sock,target=/run/host-services/ssh-auth.sock -e SSH_AUTH_SOCK="/run/host-services/ssh-auth.sock" -it --rm alpine:3.4 /bin/sh -c "apk -q update && apk -q add dropbear-ssh && ssh -T git@github.com"

Host 'github.com' is not in the trusted hosts file.
(ssh-rsa fingerprint md5 16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48)
Do you want to continue connecting? (y/n) y
Hi yuroyoro! You've successfully authenticated, but GitHub does not provide shell access.
```

この `/run/host-services/ssh-auth.sock` はDocker for Macがssh agentをコンテナにproxyするために用意するunix domain socketで、必ずこのパスを指定しなければなりません。ホスト側の `SSH_AUTH_SOCK` ではない点が注意です。

`docker-compose` の場合は、 `docker-compose.yml` に書いておくとよいようです。

```
services:
  web:
    image: nginx:alpine
    volumes:
      - type: bind
        source: /run/host-services/ssh-auth.sock
        target: /run/host-services/ssh-auth.sock
    environment:
      - SSH_AUTH_SOCK=/run/host-services/ssh-auth.sock
```

# MacOSXでUID指定する場合

で、ここからが最近ハマった点ですが、MacOSXで `docker run` する際にUID/GIDを指定して実行する場合、上記の手順を踏んでも失敗します。
原因は以下のとおりです。


<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">docker for macがマウントする/run/host-services/ssh-auth.sockはroot所有でパーミッションは755になる<br><br>UID指定時にコンテナでsshすると上記のファイルをsocket(2)で開こうとするが、書き込み権限がないためagentに接続できない<br><br>pubkey_prepare: ssh_get_authentication_socket: Permission denied</p>&mdash; いきもの (@yuroyoro) <a href="https://twitter.com/yuroyoro/status/1232904630377467904?ref_src=twsrc%5Etfw">February 27, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

対策としては、 `ENTRYPOINT` でsudoで上記のsocketファイルの権限を666にchmodすればよいです。
以下の参考にして `entry_point.sh` でユーザーを作成して `sudo chmod 666 $SSH_AUTH_SOCK` してsocketに書き込めるようにします。

[Dockerでuid/gid指定可能かつsudo実行可能なユーザとしてコンテナを起動する \- Qiita](https://qiita.com/yama07/items/a521234dc91f923ba655) 


```bash
#!/bin/bash

USER_ID=$(id -u)
GROUP_ID=$(id -g)

# create group
gent=$(getent group $GROUP_ID)
if [ -z "$gent" ]; then
    groupadd -g $GROUP_ID $USER
fi

# create user if not root
if [ x"$USER_ID" != x"0" ]; then
    useradd -d /home/$USER -m -s /bin/bash -u $USER_ID -g $GROUP_ID -G sudo $USER
fi

# allow to write SSH_AUTH_SOCK
if [ -e "$SSH_AUTH_SOCK" ]; then
  sudo chmod 666 $SSH_AUTH_SOCK
fi

# remove SUID
sudo chmod u-s /usr/sbin/useradd
sudo chmod u-s /usr/sbin/groupadd

exec $@
```

ヨシ!

