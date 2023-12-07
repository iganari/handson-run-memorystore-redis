# 違うプロジェクトの Cloud Run から Memorystore にアクセスする

## メモ

これが必要
https://cloud.google.com/run/docs/configuring/shared-vpc-direct-vpc?hl=ja

## 概要

![](./_img/diffproject-overview.png)

## 0. 準備

+ 環境変数をセットしておきます

```
export _gc_pj_id_host='hogehoge-host'
export _gc_pj_id_service='hogehoge-service'


### Different Projects cloudRun memorystoreforRedis Shared VPC
# export _common='dprrshared'

export _common='mmmmmmm'
export _region='asia-northeast1'

### 実際には 01 しか使わないが用意しておく
export _sub_network_range='10.146.0.0/20'
# export _sub_network_range_02='10.174.0.0/20'
# export _sub_network_range_03='10.178.0.0/20'
```

+ API の有効化

```
gcloud beta services enable compute.googleapis.com           --project ${_gc_pj_id_01}
gcloud beta services enable redis.googleapis.com             --project ${_gc_pj_id_01}
gcloud beta services enable servicenetworking.googleapis.com --project ${_gc_pj_id_01}
```

```
gcloud beta services enable compute.googleapis.com          --project ${_gc_pj_id_02}
gcloud beta services enable artifactregistry.googleapis.com --project ${_gc_pj_id_02}
```



https://cloud.google.com/vpc/docs/provisioning-shared-vpc

## 必要な Role

この作業者は組織レベルで以下の Role を持っている必要がある

+ Compute Shared VPC Admin (compute.organizations.enableXpnHost)

## 組織のID を取得する

スクショ


```
export _gc_org_id='62602512451'
```



## ホストでやること



## 共有 VPC のホストプロジェクトとして有効化

+ 共有 VPC のホストプロジェクトにする Google Cloud のプロジェクトに対して、共有 VPC を有効化

```
gcloud beta compute shared-vpc enable ${_gc_pj_id_host}
```

+ 共有 VPC の設定の確認

```
gcloud beta compute shared-vpc organizations list-host-projects ${_gc_org_id} --filter=${_gc_pj_id_host}
```
```
### 例
$ gcloud beta compute shared-vpc organizations list-host-projects ${_gc_org_id} --filter=${_gc_pj_id_host}
NAME                        CREATION_TIMESTAMP  XPN_PROJECT_STATUS
hogehoge-host
```

+ 共有 VPC のホストプロジェクトにサービスプロジェクトを接続

```
gcloud beta compute shared-vpc associated-projects add ${_gc_pj_id_service} \
  --host-project ${_gc_pj_id_host}
```

+ サービスプロジェクトの紐付けの確認

```
gcloud beta compute shared-vpc list-associated-resources ${_gc_pj_id_host}
```
```
### 例

$ gcloud beta compute shared-vpc list-associated-resources ${_gc_pj_id_host}
RESOURCE_ID                 RESOURCE_TYPE
hogehoge-service            PROJECT
```

## ネットワーク

+ VPC Network の作成

```
gcloud beta compute networks create ${_common}-network \
  --subnet-mode custom \
  --project ${_gc_pj_id_host}
```

+ サブネットの作成

```
gcloud beta compute networks subnets create ${_common}-subnets \
  --network ${_common}-network \
  --region ${_region} \
  --range ${_sub_network_range} \
  --enable-private-ip-google-access \
  --project ${_gc_pj_id_host}
```


### 1-2. ネットワークの作成

+ Private Services Access の設定

```
gcloud beta compute addresses create ${_common}-psa \
  --global \
  --network ${_common}-network \
  --purpose VPC_PEERING \
  --prefix-length 16 \
  --project ${_gc_pj_id_host}
```

+ Private Connection の作成

```
gcloud beta services vpc-peerings connect \
  --network ${_common}-network \
  --ranges ${_common}-psa \
  --service servicenetworking.googleapis.com \
  --project ${_gc_pj_id_host}
```

### 1-3. Memorystore for Redis

+ 環境変数を設定しておく

```
export _instance_tier='basic'
export _instance_num='1'
export _redis_ver='redis_7_0'

### Private services access の場合
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
  --network=projects/${_gc_pj_id_host}/global/networks/${_common}-network \
  --reserved-ip-range ${_common}-psa \
  --project ${_gc_pj_id_host} \
  --async
```

ちょっと待ちます :coffee:

+ Memorystore for Redis のインスタンスのエンドポイントを確認する
  + Cloud Run デプロイ時に使います

```
gcloud beta redis instances describe ${_common}-redis \
  --region ${_region} \
  --project ${_gc_pj_id_host} \
  --format json | jq -r .host
```

+ Memorystore for Redis のインスタンスのエンドポイントを環境変数に入れておく

```
export _redis_host=$(gcloud beta redis instances describe ${_common}-redis \
  --region ${_region} \
  --project ${_gc_pj_id_host} \
  --format json | jq -r .host)
export _redis_port=$(gcloud beta redis instances describe ${_common}-redis \
  --region ${_region} \
  --project ${_gc_pj_id_host} \
  --format json | jq -r .port)

### 確認
echo ${_redis_host}
echo ${_redis_port}
```


## 2. サービスプロジェクトでの設定


### 2-1. IAM

+ Cloud Run 用の Service Account を発行

```
gcloud beta iam service-accounts create ${_common}-run-sa \
  --description="${_common} の Cloud Run 用のサービスアカウント" \
  --display-name="${_common}-run-sa" \
  --project ${_gc_pj_id_service}
```

+ 上記で作成した Service Account に以下の Role を付与します
  + https://cloud.google.com/run/docs/configuring/shared-vpc-direct-vpc?hl=en#set_up_iam_permissions
  + ホストプロジェクトにて
    + プロジェクトレベルの Compute Network Viewer (compute.networkViewer)
    + サブネットレベルでの Compute Network User (compute.networkUser)
  + Cloud Run Service Agent (roles/run.serviceAgent) をサービスプロジェクト内で付与する



+ サービスプロジェクトの Project Number を取得

```
export _gc_pj_num_service=`gcloud beta projects describe ${_gc_pj_id_service} --format json | jq -r .projectNumber`

echo ${_gc_pj_num_service}
```

+ サービスプロジェクトの Google Cloud Run Service Agent の Service Account に、ホストプロジェクト内で Role を付与する
  + **service-${Project Number}@serverless-robot-prod.iam.gserviceaccount.com** の形をしている

```
### compute.networkViewer on Project
gcloud beta projects add-iam-policy-binding ${_gc_pj_id_host} \
  --member "serviceAccount:service-${_gc_pj_num_service}@serverless-robot-prod.iam.gserviceaccount.com" \
  --role "roles/compute.networkViewer"

### compute.networkUser on Subnets
gcloud beta compute networks subnets add-iam-policy-binding ${_common}-subnets-01 \
  --region ${_region} \
  --member "serviceAccount:service-${_gc_pj_num_service}@serverless-robot-prod.iam.gserviceaccount.com" \
  --role "roles/compute.networkUser" \
  --project ${_gc_pj_id_host}
```

+ サービスプロジェクト内の Cloud Run 用の Service Account が、同じプロジェクト内の Google Cloud Run Service Agent の Role を使用出来るようにする
  + Cloud Run Service Agent (roles/run.serviceAgent)

```
gcloud beta projects add-iam-policy-binding ${_gc_pj_id_service} \
  --member "serviceAccount:${_common}-run-sa@${_gc_pj_id_service}.iam.gserviceaccount.com" \
  --role "roles/run.serviceAgent"
```



### 2-2. ネットワークの作成

ここは特に無し



### 2-4. Artifact Registry のリポジトリ作成とコンテナイメージの格納

+ Artifact Registry のリポジトリを作成

```
gcloud beta artifacts repositories create ${_common}-ar \
  --repository-format docker \
  --location ${_region} \
  --project ${_gc_pj_id_service}
```

+ Cloud Run にデプロイするコンテナイメージの作成とアップロード

```
git clone https://github.com/joeferner/redis-commander.git
cd redis-commander

docker build -f Dockerfile . --tag=${_region}-docker.pkg.dev/${_gc_pj_id_service}/${_common}-ar/redis-commander:latest
docker push ${_region}-docker.pkg.dev/${_gc_pj_id_service}/${_common}-ar/redis-commander:latest
```

### 2-5. Cloud Run のサービスのデプロイ

+ Cloud Run のサービスのデプロイ
  + ${_common}-subnets-01 を使います
  https://cloud.google.com/run/docs/configuring/shared-vpc-direct-vpc?hl=ja#deploy-service

```
gcloud beta run deploy ${_common}-run \
  --platform managed \
  --network projects/${_gc_pj_id_host}/global/networks/${_common}-network \
  --subnet projects/${_gc_pj_id_host}/regions/${_region}/subnetworks/${_common}-subnets \
  --region ${_region} \
  --cpu 1000m \
  --memory 512Mi \
  --image ${_region}-docker.pkg.dev/${_gc_pj_id_service}/${_common}-ar/redis-commander:latest \
  --port 8081 \
  --set-env-vars=REDIS_HOSTS=${_redis_host}, \
  --ingress all \
  --vpc-egress private-ranges-only \
  --allow-unauthenticated \
  --min-instances 1 \
  --max-instances 1 \
  --service-account ${_common}-run-sa@${_gc_pj_id_service}.iam.gserviceaccount.com \
  --project ${_gc_pj_id_service} \
  --quiet
```

+ Cloud Run のサービスの確認

```
gcloud beta run services describe ${_common}-run \
  --region ${_region} \
  --project ${_gc_pj_id} \
  --format json
```

## 6. Web ブラウザで確認する

![](./_img/6-1.png)

## 99. クリーンアップ

<details>
<summary>99-1. Cloud Run のサービスの削除</summary>

```
gcloud beta run services delete ${_common}-run \
  --region ${_region} \
  --project ${_gc_pj_id_service}
```

</details>

<details>
<summary>99-2. Artifact Registry の削除</summary>

```
gcloud beta artifacts repositories delete ${_common}-ar \
  --location ${_region} \
  --project ${_gc_pj_id_service}
```

</details>

<details>
<summary>99-3. Memorystore for Redis のインスタンスの削除</summary>

```
gcloud beta redis instances delete ${_common}-redis \
  --region ${_region} \
  --project ${_gc_pj_id_host} \
  --async
```

</details>


<details>
<summary>99-4. Service Account から Role を剥奪</summary>

```
gcloud beta projects remove-iam-policy-binding ${_gc_pj_id_service} \
  --member "serviceAccount:${_common}-run-sa@${_gc_pj_id_service}.iam.gserviceaccount.com" \
  --role "roles/run.serviceAgent"
```

</details>


<details>
<summary>99-4. Google Cloud Run Service Agent の Service Account から Role を剥奪</summary>

```
### compute.networkUser on Subnets
gcloud beta compute networks subnets remove-iam-policy-binding ${_common}-subnets-01 \
  --region ${_region} \
  --member "serviceAccount:service-${_gc_pj_num_service}@serverless-robot-prod.iam.gserviceaccount.com" \
  --role "roles/compute.networkUser" \
  --project ${_gc_pj_id_host}

### compute.networkViewer on Project
gcloud beta projects remove-iam-policy-binding ${_gc_pj_id_host} \
  --member "serviceAccount:service-${_gc_pj_num_service}@serverless-robot-prod.iam.gserviceaccount.com" \
  --role "roles/compute.networkViewer"
```

</details>


<details>
<summary>99-4. Cloud Run 用の Service Account を削除</summary>

```
gcloud beta iam service-accounts delete ${_common}-run-sa@${_gc_pj_id_service}.iam.gserviceaccount.com \
  --project ${_gc_pj_id_service}
```

</details>









※ ここでしばらく待つ (15 分くらい)


<details>
<summary>99-4. Private Connection の削除</summary>

```
gcloud beta services vpc-peerings delete \
  --network ${_common}-network \
  --service servicenetworking.googleapis.com \
  --project ${_gc_pj_id_host}
```

</details>

<details>
<summary>99-5. Private Services Access の削除</summary>

```
gcloud beta compute addresses delete ${_common}-psa \
  --global \
  --project ${_gc_pj_id_host}
```

</details>









※ ここでしばらく待つ (1~2 時間くらい)



<details>
<summary>99-4. 共有 VPC のホストプロジェクトに接続しているサービスプロジェクトを解除</summary>

```
gcloud beta compute shared-vpc associated-projects remove ${_gc_pj_id_service} \
  --host-project ${_gc_pj_id_host}
```


</details>







<details>
<summary>99-6. サブネットの削除</summary>

```
gcloud beta compute networks subnets delete ${_common}-subnets \
  --region ${_region} \
  --project ${_gc_pj_id_host}
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

