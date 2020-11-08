# Anthos Service Mesh w/ Multiple Ingress Gateways
  ASM で複数の istio-ingressgateway をデプロイした時のメモ

オフィシャルの ASM インストール手順はこちら：  
https://cloud.google.com/service-mesh/docs/install  

## GKE クラスタのデプロイ

### 各変数の設定

```bash
export GCP_EMAIL_ADDRESS=$(gcloud auth list --filter=status:ACTIVE \
  --format="value(account)")
export PROJECT_ID=$(gcloud config list --format   "value(core.project)")
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID}   --format="value(projectNumber)")
export CLUSTER_NAME=multi-gw
export CLUSTER_LOCATION=asia-southeast1-a
export CLUSTER_REGION=asia-southeast1
export WORKLOAD_POOL=${PROJECT_ID}.svc.id.goog
export MESH_ID="proj-${PROJECT_NUMBER}"
export CHANNEL="regular"
export GCP_EMAIL_ADDRESS=$(gcloud auth list --filter=status:ACTIVE   --format="value(account)")
export ENVIRON_PROJECT_NUMBER=$PROJECT_NUMBER
export POD_CIDR="10.56.0.0/14"
export POD_CIDR_NAME="pods"
export SVCS_CIDR="10.120.0.0/20"

gcloud config set compute/zone ${CLUSTER_LOCATION}
```

### API の有効化

```bash
gcloud services enable \
    --project=${PROJECT_ID} \
    container.googleapis.com \
    compute.googleapis.com \
    monitoring.googleapis.com \
    logging.googleapis.com \
    cloudtrace.googleapis.com \
    meshca.googleapis.com \
    meshtelemetry.googleapis.com \
    meshconfig.googleapis.com \
    iamcredentials.googleapis.com \
    gkeconnect.googleapis.com \
    gkehub.googleapis.com \
    cloudresourcemanager.googleapis.com
```

### サブネットの作成

```bash
gcloud compute networks subnets update default \
    --region ${CLUSTER_REGION} \
    --add-secondary-ranges ${POD_CIDR_NAME}=${POD_CIDR}
```

### GKE クラスタのデプロイ

```bash
gcloud beta container clusters create ${CLUSTER_NAME} \
    --zone ${CLUSTER_LOCATION} \
    --enable-ip-alias \
    --machine-type e2-custom-4-6144 \
    --num-nodes=4 \
    --workload-pool=${WORKLOAD_POOL} \
    --enable-stackdriver-kubernetes \
    --subnetwork=default \
    --cluster-secondary-range-name=pods \
    --services-ipv4-cidr=${SVCS_CIDR} \
    --addons HorizontalPodAutoscaling,HttpLoadBalancing \
    --labels mesh_id=${MESH_ID} \
    --release-channel=regular
```

### Credential の取得

```bash
gcloud container clusters get-credentials ${CLUSTER_NAME} \
    --zone ${CLUSTER_LOCATION}
```

### プロジェクトの初期化

```bash
curl --request POST   --header "Authorization: Bearer $(gcloud auth print-access-token)"   --data ''   https://meshconfig.googleapis.com/v1alpha1/projects/${PROJECT_ID}:initialize
```

### GKE クラスタの Role Binding 設定 (cluster-admin)

```bash
kubectl create clusterrolebinding cluster-admin-binding   --clusterrole=cluster-admin   --user="$(gcloud config get-value core/account)"
```

## Anthos Service Mesh のインストール

### インストール資源のダウンロード

```bash
export ASM_VERSION=istio-1.6.11-asm.1
curl -LO https://storage.googleapis.com/gke-release/asm/${ASM_VERSION}-linux-amd64.tar.gz

tar xzf ${ASM_VERSION}-linux-amd64.tar.gz

cd ${ASM_VERSION}
export PATH=$PWD/bin:$PATH
```

### ASM パッケージのダウンロードと各種設定

```bash
sudo apt-get install google-cloud-sdk-kpt -y

kpt pkg get https://github.com/GoogleCloudPlatform/anthos-service-mesh-packages.git/asm@release-1.6-asm asm
kpt cfg set asm gcloud.container.cluster ${CLUSTER_NAME}
kpt cfg set asm gcloud.core.project ${PROJECT_ID}
kpt cfg set asm gcloud.compute.location ${CLUSTER_LOCATION}
kpt cfg set asm gcloud.project.environProjectNumber ${ENVIRON_PROJECT_NUMBER}
kpt cfg set asm anthos.servicemesh.profile asm-gcp
```

### ASM インストール w/ Multiple Ingress Gateway

```bash
git clone https://github.com/kkuchima/asm-multi-ingressgw

istioctl install -f asm/cluster/istio-operator.yaml \
    --revision=asm-1611-1 \
    --set values.global.proxy.tracer=stackdriver \
    --set values.pilot.traceSampling=100 \
    --set values.telemetry.v2.stackdriver.enabled=true \
    --set values.telemetry.v2.stackdriver.logging=true \
    --set values.telemetry.v2.stackdriver.outboundAccessLogging=FULL \
    --set meshConfig.outboundTrafficPolicy.mode=ALLOW_ANY \
    -f asm-multi-ingressgw/istio-configuration/multi-gateway.yaml
```

### istio-system namespace 内の Pod が Running 状態になっていることを確認

```bash
kubectl get pods -n istio-system -w

### 出力例:
NAME                                             READY   STATUS    RESTARTS   AGE
istio-ingressgateway-6fd88bf4df-vnrgp            1/1     Running   0          48m
istio-internal-ingressgateway-79745cbc75-gnbqx   1/1     Running   0          48m
istiod-asm-1611-1-85b6bc4f4-rpq5j                1/1     Running   1          48m
istiod-asm-1611-1-85b6bc4f4-xsb7t                1/1     Running   0          48m
```

### 対象 Namespace に対して envoy auto-injection を有効化

```bash
kubectl label namespace default istio-injection- istio.io/rev=asm-1611-1 --overwrite
```

### サンプルアプリのデプロイ

```bash
### External アクセス用 (httpbin)
kubectl apply -f asm-multi-ingressgw/sample-app/

### External アクセス用 (nginx)
kubectl apply -f asm-multi-ingressgw/sample-app-internal/
```

## 疎通テスト

### 各 Istio-ingressgateway Service の IP アドレス を確認

```bash
kubectl get svc -n istio-system

### 出力例:
NAME                            TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                                                      AGE
istio-ingressgateway            LoadBalancer   10.120.6.54     35.247.145.10   15021:30126/TCP,80:30967/TCP,443:30622/TCP,15443:30874/TCP   45m
istio-internal-ingressgateway   LoadBalancer   10.120.8.43     10.148.0.26     15020:30607/TCP,80:32700/TCP,443:30707/TCP                   45m
istiod-asm-1611-1               ClusterIP      10.120.11.245   <none>          15010/TCP,15012/TCP,443/TCP,15014/TCP,853/TCP                45m
```


### 外部IPアドレス到達可能な端末
外部ネットワークに到達可能な端末から、Istio-ingressgateway (External LB) の IP アドレス宛てに curl を発行する。
```bash

curl 35.247.145.10
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <title>httpbin.org</title>
    <link href="https://fonts.googleapis.com/css?family=Open+Sans:400,700|Source+Code+Pro:300,600|Titillium+Web:400,600,700"
        rel="stylesheet">
    <link rel="stylesheet" type="text/css" href="/flasgger_static/swagger-ui.css">
    <link rel="icon" type="image/png" href="/static/favicon.ico" sizes="64x64 32x32 16x16" />
    <style>
        html {
            box-sizing: border-box;
            overflow: -moz-scrollbars-vertical;
            overflow-y: scroll;
        }

        *,
        *:before,
        *:after {
            box-sizing: inherit;
        }

        body {
            margin: 0;
            background: #fafafa;
        }
    </style>
</head>
```

### 同一 VPC 内端末
同一VPCの同一リージョン内端末から、 Istio-ingressgateway (Internal LB) の IP アドレス宛てに curl を発行する。
```bash
curl 10.148.0.26
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
