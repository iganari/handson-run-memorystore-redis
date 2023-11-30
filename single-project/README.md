# 同じプロジェクト内で Cloud Run から Memorystore にアクセスする


## 0. 準備

環境変数的な何か

```
export _gc_pj_id='Your Google Cloud ID'
export _common='singlerunredis'
export _sub_network_range='10.146.0.0/20'
export _region='asia-northeast1'
```

## 1. IAM

+ Cloud Run 用の Service Account を発行し、 Cloud SQL に接続できる最低限の Role を付与する

```
gcloud beta iam service-accounts create ${_common}-run-sa \
  --description="Cloud Run 用のサービスアカウント" \
  --display-name="${_common}-run-sa" \
  --project ${_gc_pj_id}
```

+ 上記で作成した Service Account に以下の Role を付与します
  + **WIP**
  + Role: Cloud SQL Client( roles/cloudsql.client )
  + Cloud SQL | Connect from Cloud Run

```
# gcloud beta projects add-iam-policy-binding ${_gc_pj_id} \
#   --member="serviceAccount:${_common}-run-sa@${_gc_pj_id}.iam.gserviceaccount.com" \
#   --role="roles/cloudsql.client"
```

## 2. ネットワーク

+ VPC Network の作成

```
gcloud beta compute networks create ${_common}-network \
  --subnet-mode=custom \
  --project ${_gc_pj_id}
```

+ Private Services Access の設定

```
gcloud beta compute addresses create ${_common}-psa \
  --global \
  --network ${_common}-network \
  --purpose VPC_PEERING \
  --prefix-length 16 \
  --project ${_gc_pj_id}
```

+ Private Connection の作成

```
gcloud beta services vpc-peerings connect \
  --network ${_common}-network \
  --ranges ${_common}-psa \
  --service servicenetworking.googleapis.com \
  --project ${_gc_pj_id}
```

+ サブネットの作成
  + Direct VPC egress 用

```
gcloud beta compute networks subnets create ${_common}-subnets \
  --network ${_common}-network \
  --region ${_region} \
  --range ${_sub_network_range} \
  --enable-private-ip-google-access \
  --project ${_gc_pj_id}
```

+ Firewall Rule の作成

```
### 内部通信は全部許可する
gcloud beta compute firewall-rules create ${_common}-allow-internal-all \
  --network ${_common}-network \
  --direction=INGRESS \
  --action ALLOW \
  --rules tcp:0-65535,udp:0-65535,icmp \
  --source-ranges ${_sub_network_range} \
  --priority=1000 \
  --project ${_gc_pj_id}

```

## 3. Memorystore for Redis

+ 環境変数を設定しておく

```
export _instance_tier='basic'
export _instance_num='1'
export _instance_region='asia-northeast1'
export _redis_ver='redis_7_0'

# Private services access の場合
export _connect_mode='PRIVATE_SERVICE_ACCESS'
```

+ Memorystore for Redis のインスタンスの作成

```
gcloud beta redis instances create ${_common}-redis \
  --tier ${_instance_tier} \
  --size ${_instance_num} \
  --redis-version ${_redis_ver} \
  --region ${_instance_region} \
  --connect-mode ${_connect_mode} \
  --network=projects/${_gc_pj_id}/global/networks/${_common}-network \
  --reserved-ip-range ${_common}-psa \
  --project ${_gc_pj_id}
```

## 4. Artifact Registry のリポジトリ作成とコンテナイメージの格納

+ Artifact Registry のリポジトリを作成

```
gcloud beta artifacts repositories create ${_common}-ar \
  --repository-format docker \
  --location ${_region} \
  --project ${_gc_pj_id}
```

+ Cloud Run にデプロイするコンテナイメージの作成とアップロード

```
git clone https://github.com/joeferner/redis-commander.git
cd redis-commander

docker build -f Dockerfile . --tag=${_region}-docker.pkg.dev/${_gc_pj_id}/${_common}-ar/redis-commander:latest
docker push ${_region}-docker.pkg.dev/${_gc_pj_id}/${_common}-ar/redis-commander:latest
```

## 5. Cloud Run のサービスのデプロイ

```
gcloud beta run deploy ${_common}-run \
  --platform managed \
  --network ${_common}-network \
  --subnet ${_common}-subnets \
  --region ${_region} \
  --cpu 1000m \
  --memory 512Mi \
  --image ${_region}-docker.pkg.dev/${_gc_pj_id}/${_common}-ar/redis-commander:latest \
  --port 8081 \
  --set-env-vars=REDIS_HOSTS=10.138.0.3, \
  --ingress all \
  --allow-unauthenticated \
  --min-instances 1 \
  --max-instances 1 \
  --service-account ${_common}-run-sa@${_gc_pj_id}.iam.gserviceaccount.com \
  --project ${_gc_pj_id} \
  --quiet
```

## 6. Web ブラウザで確認する



## 99. クリーンアップ

TBD

## VM からのデバック

```
redis-cli -h 10.138.0.3 -p 6379 PING
---> OK
```


```
SET mykey "Hello\nWorld"
GET mykey

--> OK
```

docker run --rm --name redis-commander -d -p 80:8081 \
  --env REDIS_HOSTS=10.138.0.3 \
  ghcr.io/joeferner/redis-commander:latest

---> アプリは 8081 で動いている