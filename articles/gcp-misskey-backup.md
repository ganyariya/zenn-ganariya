---
title: "GCPで Misskey の PostgresDB とメディアファイルをバックアップする"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gcp", "misskey"]
published: true
---

# はじめに

https://zenn.dev/ganariya/articles/gcp-misskey-instance

https://misskey.ganyariya.net/

GCP の Google Compute Engine で Misskey インスタンスを作成しました。
しかし、上記で構築したインスタンスは以下のように建てられています。

- e2-micro (無料枠) という弱いスペック
- postgre は直接 VM 上で動いている
  - redis も同様に VM 上で動いている
- 画像や音声などのメディアファイルは VM 上の files というディレクトリに直接保存されている
  - VM 上の misskey が直接配信している

VM 上に直接 postgreSQL やメディアファイルが乗っているため、 VM を消すとデータがすべて消えます。

https://hide.ac/articles/E2Ea3cauk

そのため、今回は GCP 上にこれらのバックアップを取得したため、その内容をまとめます。

# postgre のバックアップ

https://hide.ac/articles/E2Ea3cauk

上記を参考に実行します。

## postgre dump

pg_dumpall で dump します。

```bash
sudo -u postgres pg_dumpall | gzip -c > misskey.gz
```

## gsutil

gsutil で GCS にアップロードします。

```bash
gsutil cp misskey.gz gs://your-storage/misskey.gz
```

このとき、 IAM ならびに VM のアクセススコープが適切に設定されているか確認が必要です。

### IAM

VM に紐付いているサービスアカウントが、アップロードしたい GCS のロール/権限を持っている必要があります。
該当のサービスアカウントに `Storage オブジェクト作成者` ロールなどを付与し、 Cloud Storage へアップロードできるようにしましょう。

```bash
# サービスアカウントの表示（どのサービスアカウントに付与すればいいかの確認）
gcloud auth list
```

### アクセススコープ

![](https://storage.googleapis.com/zenn-user-upload/9db5ffd3d790-20230325.png)

https://blog.g-gen.co.jp/entry/vm-api-iam

自分の場合、 IAM の設定は問題なかったのですが、アクセススコープの設定が足りておらずうまくアップロードできていませんでした。
VM から GCS の API を呼び出すために、アクセススコープに `ストレージ 読み取り / 書き込み` を設定しました。

（本来アクセススコープはすべての API を許可し、かつサービスアカウントに最小の権限のみ与えるのがよいのかなと思います。）

## cron 設定

```bash
sudo mkdir /var/backup
sudo chmod 777 /var/backup

sudo vi /etc/cron.d/pgbuckup

# pgbuckup 毎週一回バックアップ
SHELL=/bin/bash
0 0 * * 1 postgres pg_dumpall | gzip -c > /var/backup/misskey.gz
0 1 * * 1 ganyariya gsutil cp /var/backup/misskey.gz gs://your-storage/misskey.gz

# 有効化
sudo chmod 644 /etc/cron.d/pgbuckup
sudo systemctl restart cron
```

# メディアファイル

## GCS の設定

![](https://storage.googleapis.com/zenn-user-upload/031c1b6309d2-20230325.png)

misskey から Cloud Storage にアップロードできるように、 `相互運用性` のページでサービスアカウント HMAC 認証を設定します。
任意のサービスアカウント（例: misskey-service-account@your-project.iam.gserviceaccount.com）を作成します。
すると、アクセスキーとシークレットキーが発行されるため、のちほど misskey のコントロールパネルで設定します。

続いて、作成したサービスアカウントに対して、 IAM ロール `Storage オブジェクト作成者` を付与してください。
misskey が画像を Cloud Storage へアップロードするために必要です。

![](https://storage.googleapis.com/zenn-user-upload/1c19dea5c1ac-20230325.png)

## Misskey の設定

https://your-misskey/admin/object-storage

misskey の object-storage を開いて、 GCS にメディアがアップロードされるように設定します。

![](https://storage.googleapis.com/zenn-user-upload/2b797b804c3d-20230325.png)

# 最後に

バックアップができたので、ひとまず VM が落ちても安心ですね。
（ただ、復旧作業は大変そうだなぁ...）

本当は Cloud SQL や MemoryStore, GKE などへの移行もしたほうがよいのでしょうが、お金がかかってしまうのでしばらくは e2-micro オンリーで運用しようと思います。
