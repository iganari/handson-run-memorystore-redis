# 違うプロジェクトの Cloud Run から Memorystore にアクセスする for VPC Network Peering 編

## 概要

絵

## 0. 準備

+ 環境変数をセットしておきます

```
export _gc_pj_id_01='ca-igarashi-test-unite-cmn'
export _sub_network_range_01='10.146.0.0/20' ## 不要かもしれない

export _gc_pj_id_02='ca-igarashi-test-unite-dev'
export _sub_network_range_02='10.146.0.0/20'

export _common='dprrpeer'
### Different Projects cloudRun memorystoreforRedis PEERing


export _region='asia-northeast1'
```


```
gcloud beta services enable compute.googleapis.com --project ${_gc_pj_id_01}
gcloud beta services enable redis.googleapis.com --project ${_gc_pj_id_01}
gcloud beta services enable servicenetworking.googleapis.com --project ${_gc_pj_id_01}
```

```
gcloud beta services enable compute.googleapis.com --project ${_gc_pj_id_02}
gcloud beta services enable artifactregistry.googleapis.com --project ${_gc_pj_id_02}
```





## 1. 01 環境の設定



### 1-1. IAM

TBD

### 1-2. ネットワークの作成

+ VPC Network の作成

```
gcloud beta compute networks create ${_common}-network-01 \
  --subnet-mode=custom \
  --project ${_gc_pj_id_01}
```

+ Private Services Access の設定

```
gcloud beta compute addresses create ${_common}-psa \
  --global \
  --network ${_common}-network-01 \
  --purpose VPC_PEERING \
  --prefix-length 16 \
  --project ${_gc_pj_id_01}
```

+ Private Connection の作成

```
gcloud beta services vpc-peerings connect \
  --network ${_common}-network-01 \
  --ranges ${_common}-psa \
  --service servicenetworking.googleapis.com \
  --project ${_gc_pj_id_01}
```

### 1-3. Memorystore for Redis

+ 環境変数を設定しておく

```
export _instance_tier='basic'
export _instance_num='1'
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
  --region ${_region} \
  --connect-mode ${_connect_mode} \
  --network=projects/${_gc_pj_id_01}/global/networks/${_common}-network-01 \
  --reserved-ip-range ${_common}-psa \
  --project ${_gc_pj_id_01} \
  --async
```

ちょっと待ちます :coffee:

+ Memorystore for Redis のインスタンスのエンドポイントを確認する
  + Cloud Run デプロイ時に使います

```
gcloud beta redis instances describe ${_common}-redis \
  --region ${_region} \
  --project ${_gc_pj_id_01} \
  --format json | jq -r .host
```



## 2. 02 環境の設定


### 2-1. IAM

+ Cloud Run 用の Service Account を発行し、 Cloud SQL に接続できる最低限の Role を付与する

```
gcloud beta iam service-accounts create ${_common}-run-sa \
  --description="Cloud Run 用のサービスアカウント" \
  --display-name="${_common}-run-sa" \
  --project ${_gc_pj_id_02}
```

+ 上記で作成した Service Account に以下の Role を付与します
  + 今回は不要

```
# 不要
```

### 2-2. ネットワークの作成

+ VPC Network の作成

```
gcloud beta compute networks create ${_common}-network-02 \
  --subnet-mode=custom \
  --project ${_gc_pj_id_02}
```

+ サブネットの作成
  + Direct VPC egress 用

```
gcloud beta compute networks subnets create ${_common}-subnets-02 \
  --network ${_common}-network-02 \
  --region ${_region} \
  --range ${_sub_network_range} \
  --enable-private-ip-google-access \
  --project ${_gc_pj_id_02}
```


### Peering

https://cloud.google.com/vpc/docs/using-vpc-peering

両方でやる

```
gcloud beta compute networks peerings create ${_common}-peering \
  --network ${_common}-network-01 \
  --peer-project ${_gc_pj_id_02} \
  --peer-network ${_common}-network-02 \
  --project ${_gc_pj_id_01}
```
```
gcloud beta compute networks peerings create ${_common}-peering \
  --network ${_common}-network-02 \
  --peer-project ${_gc_pj_id_01} \
  --peer-network ${_common}-network-01 \
  --project ${_gc_pj_id_02}
```

ピアリング接続のルートを一覧表示する

```
### incoming
gcloud beta compute networks peerings list-routes ${_common}-peering \
  --network ${_common}-network-01 \
  --region ${_region} \
  --direction incoming \
  --project ${_gc_pj_id_01}

### outgoing
gcloud beta compute networks peerings list-routes ${_common}-peering \
  --network ${_common}-network-01 \
  --region ${_region} \
  --direction outgoing \
  --project ${_gc_pj_id_01}
```
```
### incoming
gcloud beta compute networks peerings list-routes ${_common}-peering \
  --network ${_common}-network-02 \
  --region ${_region} \
  --direction incoming \
  --project ${_gc_pj_id_02}

### outgoing
gcloud beta compute networks peerings list-routes ${_common}-peering \
  --network ${_common}-network-02 \
  --region ${_region} \
  --direction outgoing \
  --project ${_gc_pj_id_02}
```








## 4. Artifact Registry のリポジトリ作成とコンテナイメージの格納

+ Artifact Registry のリポジトリを作成

```
gcloud beta artifacts repositories create ${_common}-ar \
  --repository-format docker \
  --location ${_region} \
  --project ${_gc_pj_id_02}
```

+ Cloud Run にデプロイするコンテナイメージの作成とアップロード

```
git clone https://github.com/joeferner/redis-commander.git
cd redis-commander

docker build -f Dockerfile . --tag=${_region}-docker.pkg.dev/${_gc_pj_id_02}/${_common}-ar/redis-commander:latest
docker push ${_region}-docker.pkg.dev/${_gc_pj_id_02}/${_common}-ar/redis-commander:latest
```

## 5. Cloud Run のサービスのデプロイ

+ Memorystore for Redis のインスタンスのエンドポイントを環境変数にいれる

```
export _redis_host=$(gcloud beta redis instances describe ${_common}-redis \
  --region ${_region} \
  --project ${_gc_pj_id_01} \
  --format json | jq -r .host)
```

+ Cloud Run のサービスのデプロイ

```
gcloud beta run deploy ${_common}-run \
  --platform managed \
  --network ${_common}-network-02 \
  --subnet ${_common}-subnets-02 \
  --region ${_region} \
  --cpu 1000m \
  --memory 512Mi \
  --image ${_region}-docker.pkg.dev/${_gc_pj_id_02}/${_common}-ar/redis-commander:latest \
  --port 8081 \
  --set-env-vars=REDIS_HOSTS=${_redis_host}, \
  --ingress all \
  --allow-unauthenticated \
  --min-instances 1 \
  --max-instances 1 \
  --service-account ${_common}-run-sa@${_gc_pj_id_02}.iam.gserviceaccount.com \
  --project ${_gc_pj_id_02} \
  --quiet
```

+ Cloud Run のサービスの確認

```
gcloud beta run services describe ${_common}-run \
  --region ${_region} \
  --project ${_gc_pj_id} \
  --format json
```



### ここまで

これだけだと繋がらないので、さらに設定をする必要がある

まずは Firewall ?







## 6. Web ブラウザで確認する

![](./_img/6-1.png)

## 99. クリーンアップ

<details>
<summary>99-1. Cloud Run のサービスの削除</summary>

```
gcloud beta run services delete ${_common}-run \
  --region ${_region} \
  --project ${_gc_pj_id}
```

</details>

<details>
<summary>99-2. Artifact Registry の削除</summary>

```
gcloud beta artifacts repositories delete ${_common}-ar \
  --location ${_region} \
  --project ${_gc_pj_id}
```

</details>

<details>
<summary>99-3. Memorystore for Redis のインスタンスの削除</summary>

```
gcloud beta redis instances delete ${_common}-redis \
  --region ${_region} \
  --project ${_gc_pj_id} \
  --async
```

</details>

<details>
<summary>99-4. Private Connection の削除</summary>

```
gcloud beta services vpc-peerings delete \
  --network ${_common}-network \
  --service servicenetworking.googleapis.com \
  --project ${_gc_pj_id}
```

</details>

<details>
<summary>99-5. Private Services Access の削除</summary>

```
gcloud beta compute addresses delete ${_common}-psa \
  --global \
  --project ${_gc_pj_id}
```

</details>

※ ここでしばらく待つ (1時間くらい)

<details>
<summary>99-6. サブネットの削除</summary>

```
gcloud beta compute networks subnets delete ${_common}-subnets \
  --region ${_region} \
  --project ${_gc_pj_id}
```

</details>

<details>
<summary>99-7. VPC Network の削除</summary>

```
gcloud beta compute networks delete ${_common}-network \
  --project ${_gc_pj_id}
```

</details>

<details>
<summary>99-8. Cloud Run 用の Service Account の削除</summary>

```
gcloud beta iam service-accounts delete ${_common}-run-sa@${_gc_pj_id}.iam.gserviceaccount.com \
  --project ${_gc_pj_id}
```

</details>

## 結び

Have Fan!! :)

