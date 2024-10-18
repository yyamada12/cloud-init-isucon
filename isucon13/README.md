# cloud-init-isucon/isucon13

## Overview

isucon13とほぼ同じ環境を構築するためのcloud-configです。

## Requirements

* Ubuntu 22.04 LTSを用意してください。

## Usage

構築手順は[トップのREADME](../README.md)をご確認ください。

## Bench

* 構築が終わったらベンチマークを実行します
  ```sh
  sudo -i -u isucon
  ./bench run --enable-ssl
  ```

構築時のタイミングによってpowerdnsの初期化に失敗してベンチマークが正しく実行されない場合があるようです。
以下を実行することで解消すると思われます。

```sh
webapp/pdns/init_zone.sh
```

## 本来の設定と異なるところ

* 本番ではドメインとして `*.u.isucon.dev` が使われていましたが、[devトップレベルドメインはHSTS preload-listに含まれており](https://ja.wikipedia.org/wiki/.dev)、正規のSSL証明書がないとアクセスできないため `*.u.isucon.local` に書き換えています
* SSL証明書は自己署名のものを用意しています
* コンテスト実施時のインスタンスタイプはc5.large(2vCPU, 4GBメモリー)が3台構成です

## FAQ

[トップのREADME](../README.md)のFAQもご確認ください

### プログラムの動かし方がわからない

以下をご確認ください。

* [ISUCON13 レギュレーション](https://isucon.net/archives/57768216.html)
* [ISUCON13 当日マニュアル](https://github.com/isucon/isucon13/blob/c52b359fc6e733e1193ac8e9835bea23856566e7/docs/cautionary_note.md)
* [ISUCON13 アプリケーションマニュアル](https://github.com/isucon/isucon13/blob/c52b359fc6e733e1193ac8e9835bea23856566e7/docs/isupipe.md)
* [ISUCON13 リポジトリ](https://github.com/isucon/isucon13)

### ブラウザで動作確認ができない

手元のPCのhostsファイルに以下を追記してください。

```
${サーバのIPアドレス} pipe.u.isucon.local
```

追記したらブラウザから `https://pipe.u.isucon.local/` にアクセスしてみてください。
アクセスすると証明書エラーが発生する可能性があります。証明書は `/etc/nginx/tls/` 配下にあるので手元の証明書ストアに登録することで回避できるはずです。


## 複数台構成

- 以下のコマンドで webapp x3, bench x1 を起動する
```
multipass launch --name isucon13-0 --cpus 2 --disk 20G --memory 4G --timeout 86400 --cloud-init isucon13/app.cfg 22.04
multipass launch --name isucon13-1 --cpus 2 --disk 20G --memory 4G --timeout 86400 --cloud-init isucon13/app.cfg 22.04
multipass launch --name isucon13-2 --cpus 2 --disk 20G --memory 4G --timeout 86400 --cloud-init isucon13/app.cfg 22.04
multipass launch --name isucon13-bench --cpus 2 --disk 20G --memory 8G --timeout 86400 --cloud-init isucon13/bench.cfg 22.04
```

※ 本番の bench は 8vCPU 

- DNSの設定が127.0.0.1 を返すようになっているので、 multipass ls で表示される ip を返すようにしてあげる(benchを向ける先のwebappだけで良い)
  - env.sh の ISUCON13_POWERDNS_SUBDOMAIN_ADDRESS を 127.0.0.1 から server の ip に書き換える (ex 192.168.64.1)
  - isucon13マニュアルの通りに、DNS設定を更新する。 ただし u.isucon.dev ではなく u.isucon.local とする必要がある

```
multipass shell isucon13-0
sudo su - isucon
vim ./env.sh # ISUCON13_POWERDNS_SUBDOMAIN_ADDRESSを変更する

pdnsutil delete-zone u.isucon.local
sudo rm -f /opt/aws-env-isucon-subdomain-address.sh.lock
sudo reboot
```

- bench 実行
```
multipass shell isucon13-bench
 ./bench run --enable-ssl --nameserver=${serverのIP}
```

