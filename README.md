# Hands On Cloud Run to Memorystore for Redis

Cloud Run から Memorystore for Redis に繋げるハンズオンです

いくつかのパターンを用意しました :)

## 1. [同じプロジェクト内で Cloud Run から Memorystore にアクセスする](./single-project/)

Memorystore for Redis の接続方法は、使用シナリオによって選ぶ必要があります

[公式ドキュメント| Memorystore for Redis/Networking/Choosing a connection mode](https://cloud.google.com/memorystore/docs/redis/networking#choosing_a_connection_mode)

### 1-1. [Direct peering](./single-project/direct-peering/)

![](./single-project/direct-peering/_img/dp-overview.png)

### 1-2. [Private service access](./single-project/private-service-access/)

![](./single-project/private-service-access/_img/psa-overview.png)

## 2. [違うプロジェクトの Cloud Run から Memorystore にアクセスする](./different-projects/)

![](./different-projects/_img/diffproject-overview.png)

## 注意点

1. ハンズオン内の情報は作成当時 (2023/12) のものとなります
1. [Direct VPC](https://cloud.google.com/run/docs/configuring/vpc-direct-vpc) egress は 2023/12 の時点では Preview の機能です

## 最後に

Have Fan!! :)
