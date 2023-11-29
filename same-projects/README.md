# 同じプロジェクト内で Cloud Run から Memorystore にアクセスする


## 0. 準備

環境変数的な何か

```
export _gc_pj_id='Your Google Cloud ID'
export _common='samerunredis'
```

## 1. IAM

WIP

+ Cloud Run 用の Service Account を発行し、 Cloud SQL に接続できる最低限の Role を付与する

```
gcloud beta iam service-accounts create ${_common}-run-sa \
  --description="Cloud Run 用のサービスアカウント" \
  --display-name="${_common}-sa" \
  --project ${_gc_pj_id}
```

+ 上記で作成した Service Account に以下の Role を付与します
  + Role: Cloud SQL Client( roles/cloudsql.client )
  + Cloud SQL | Connect from Cloud Run

```
# gcloud beta projects add-iam-policy-binding ${_gc_pj_id} \
#   --member="serviceAccount:${_common}-run-sa@${_gc_pj_id}.iam.gserviceaccount.com" \
#   --role="roles/cloudsql.client"
```

## 2. ネットワーク

VPC

## 3. Memorystore for Redis

インスタンス

## 4. Cloud Run

https://github.com/ErikDubbelboer/phpRedisAdmin

## 99. クリーンアップ

TBD
