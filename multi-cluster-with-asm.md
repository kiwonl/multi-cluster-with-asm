## 1. 멀티 클러스터 데모를 위한 환경 변수 설정
- 단일 프로젝트 환경 에서 서로 다른 Region 에 있는 두 GKE 클러스터

   ```
   # CLUSTER-1
   export PROJECT_1=kwlee-goog-sandbox
   export CLUSTER_1=asm-multi-neg-1
   export LOCATION_1=us-central1-c
   
   # CLUSTER-2
   export PROJECT_2=kwlee-goog-sandbox
   export CLUSTER_2=asm-multi-neg-2
   export LOCATION_2=asia-northeast1-c

   # namespace for application deploy
   export NAMESPACE=sample
   ```

## 2. Cluster_1 생성과 애플리케이션 배포(whereami)
- GKE 클러스터 생성 ${CLUSTER_1}
   - 클러스터 생성 시, [ASM 설치를 위한 클러스터 요구 사항](https://cloud.google.com/service-mesh/docs/unified-install/anthos-service-mesh-prerequisites#cluster_requirements) 참조 (Workload Identity, vCPU 4개 이상인 머신, 클러스터에 최소 8개의 vCPU 등)

   ```
   gcloud container clusters create ${CLUSTER_1} \
    --project=${PROJECT_1} \
    --zone=${LOCATION_1} \
    --machine-type=e2-standard-4 \
    --num-nodes=3 \
    --workload-pool=${PROJECT_1}.svc.id.goog
   ```
- 생성한 클러스터의 인증정보와 엔드포인트 정보를 kubeconfig에 업데이트
   ```
   gcloud container clusters get-credentials ${CLUSTER_1} \
    --project=${PROJECT_1} \
    --zone=${LOCATION_1}
    
   export CTX_1="gke_${PROJECT_1}_${LOCATION_1}_${CLUSTER_1}"
   ```

- namespace 생성, 애플리케이션 (whereami) 배포
  - 두 클러스터에 동일한 namespace를 생성하고, 동일 namespace 에 애플리케이션을 배포함. [Namespace sameness](https://cloud.google.com/anthos/multicluster-management/fleets#namespace_sameness)
   ```
   kubectl create --context=${CTX_1} namespace ${NAMESPACE}
   kubectl --context=${CTX_1} apply -f ./kube/cluster.yaml --namespace ${NAMESPACE}
   ```

- 배포한 애플리케이션 동작 테스트

   ***모든 트래픽은 클러스터 내의 Pod로만 전달됨***
   ```
   $ kubectl get po,svc --context=${CTX_1} --namespace ${NAMESPACE}
   NAME                                       READY   STATUS        RESTARTS   AGE
   pod/whereami-deployment-86bc7496d8-86pxc   1/1     Running   0          6m34s
   pod/whereami-deployment-86bc7496d8-9dffb   1/1     Running   0          29m

   NAME                       TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
   service/whereami-service   ClusterIP   10.24.8.127   <none>        80/TCP    76s

   $ kubectl --context=${CTX_1} --namespace ${NAMESPACE} exec pod/whereami-deployment-86bc7496d8-9dffb -it -- /bin/sh
   $ curl whereami-service.sample.svc.cluster.local
   {
     "cluster_name": "asm-multi-neg-1",
     "host_header": "whereami-service.sample.svc.cluster.local",
     "pod_name": "whereami-deployment-86bc7496d8-86pxc",
     "pod_name_emoji": "🇹🇴",
     "project_id": "kwlee-goog-sandbox",
     "timestamp": "2022-04-28T05:36:20",
     "zone": **"us-central1-c"**
   }
   $ curl whereami-service.sample.svc.cluster.local
   {
     "cluster_name": "asm-multi-neg-1",
     "host_header": "whereami-service.sample.svc.cluster.local",
     "pod_name": "whereami-deployment-86bc7496d8-9dffb",
     "pod_name_emoji": "👨⚖️",
     "project_id": "kwlee-goog-sandbox",
     "timestamp": "2022-04-28T05:36:21",
     "zone": "us-central1-c"
   }
   ```

## 3. Cluster_2 생성과 애플리케이션 배포(whereami)
- GKE 클러스터 생성 ${CLUSTER_2}
   ```
   gcloud container clusters create ${CLUSTER_2} \
    --project=${PROJECT_2} \
    --zone=${LOCATION_2} \
    --machine-type=e2-standard-4 \
    --num-nodes=3 \
    --workload-pool=${PROJECT_2}.svc.id.goog
   ```
   ```
   gcloud container clusters get-credentials ${CLUSTER_2} \
    --project=${PROJECT_2} \
    --zone=${LOCATION_2}

   export CTX_2="gke_${PROJECT_2}_${LOCATION_2}_${CLUSTER_2}"
   ```
   ```
   kubectl create --context=${CTX_2} namespace ${NAMESPACE}
   kubectl --context=${CTX_2} apply -f ./kube/cluster-2.yaml --namespace ${NAMESPACE}
   ```
- 배포한 애플리케이션 동작 테스트

   ***모든 트래픽은 클러스터 내의 Pod로만 전달됨***
   ```
   $ kubectl get po,svc --context=${CTX_2} --namespace ${NAMESPACE}
   NAME                                       READY   STATUS    RESTARTS   AGE
   pod/whereami-deployment-86bc7496d8-m2knq   1/1     Running   0          21m
   pod/whereami-deployment-86bc7496d8-xlsxh   1/1     Running   0          8m40s
   
   NAME                       TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
   service/whereami-service   ClusterIP   10.88.11.39   <none>        80/TCP    20s
      
   $ kubectl --context=${CTX_2} --namespace ${NAMESPACE} exec pod/whereami-deployment-86bc7496d8-m2knq -it -- /bin/sh
   $ curl whereami-service.sample.svc.cluster.local
   {
     "cluster_name": "asm-multi-neg-2",
     "host_header": "whereami-service.sample.svc.cluster.local",
     "pod_name": "whereami-deployment-86bc7496d8-xlsxh",
     "pod_name_emoji": "💑🏾",
     "project_id": "kwlee-goog-sandbox",
     "timestamp": "2022-04-28T05:39:37",
     "zone": "asia-northeast1-c"
   }
   $ curl whereami-service.sample.svc.cluster.local
   {
     "cluster_name": "asm-multi-neg-2",
     "host_header": "whereami-service.sample.svc.cluster.local",
     "pod_name": "whereami-deployment-86bc7496d8-m2knq",
     "pod_name_emoji": "👨🏾⚕️",
     "project_id": "kwlee-goog-sandbox",
     "timestamp": "2022-04-28T05:39:38",
     "zone": "asia-northeast1-c"
   }
   ```

## 4. ASM 설치
- [download asmcli](https://cloud.google.com/service-mesh/docs/unified-install/install-dependent-tools#download_asmcli) to install ASM
   ```
   curl https://storage.googleapis.com/csm-artifacts/asm/asmcli_1.13 > asmcli
   chmod +x asmcli
   ```
- install ASM to ${CLUSTER_1}.
   [macOS isn't supported for installation ASM](https://cloud.google.com/service-mesh/docs/unified-install/get-started#install_required_tools)
   Also, the ingress gateway is not installed now.
   ```
   ./asmcli install \
   --project_id ${PROJECT_1} \
   --cluster_name ${CLUSTER_1} \
   --cluster_location ${LOCATION_1} \
   --output_dir ./anthos-service-mesh \
   --enable_all  \
   --ca mesh_ca
   ```
- install ASM to ${CLUSTER_2}.
   ```
   ./asmcli install \
   --project_id ${PROJECT_2} \
   --cluster_name ${CLUSTER_2} \
   --cluster_location ${LOCATION_2} \
   --output_dir ./anthos-service-mesh \
   --enable_all  \
   --ca mesh_ca
   ```
- [Injecting sidecar proxies](https://cloud.google.com/service-mesh/docs/proxy-injection)
   ```
   export REVISION=$(kubectl get deploy -n istio-system -l app=istiod -o jsonpath={.items[*].metadata.labels.'istio\.io\/rev'}'{"\n"}')
   ## REVISION=asm-1132-2

   kubectl --context=${CTX_1} label namespace ${NAMESPACE} istio-injection- istio.io/rev=${REVISION} --overwrite
   kubectl --context=${CTX_1} rollout restart deployment whereami-deployment --namespace ${NAMESPACE}
   kubectl --context=${CTX_2} label namespace ${NAMESPACE} istio-injection- istio.io/rev=${REVISION} --overwrite
   kubectl --context=${CTX_2} rollout restart deployment whereami-deployment --namespace ${NAMESPACE}
   ```

-  Envoy Proxy 설치 확인
 
  ***각 pod 마다 container 가 2개씩 (main container + sidecar) 생성된 것 확인***
   ```
   $ kubectl get po,svc --context=${CTX_1} --namespace ${NAMESPACE}
   NAME                                       READY   STATUS    RESTARTS   AGE
   pod/whereami-deployment-5755d8b68b-kx4ss   2/2     Running   0          2m9s
   pod/whereami-deployment-5755d8b68b-kxzzx   2/2     Running   0          2m12s

   $ kubectl get po,svc --context=${CTX_2} --namespace ${NAMESPACE}
   NAME                                       READY   STATUS    RESTARTS   AGE
   pod/whereami-deployment-764cbfccdb-dw8ct   2/2     Running   0          2m2s
   pod/whereami-deployment-764cbfccdb-vlzfg   2/2     Running   0          2m12s   
   ```

## 5. istio-ingrssgateway 설치, ${CLUSTER-1}에만

- [Install an ingress gateway](https://cloud.google.com/service-mesh/docs/gateways#deploy_gateways)
  - gateway 는 기본 설치가 아니기 때문에 ASM설치 이후, 별도 설치해야 함
  - [설치 모범 사례 참조](https://cloud.google.com/service-mesh/docs/gateways#best_practices_for_deploying_gateways)
   ```
   export GATEWAY_NAMESPACE=istio-ingress

   kubectl create namespace ${GATEWAY_NAMESPACE} --context=${CTX_1} 
   kubectl --context=${CTX_1} label namespace ${GATEWAY_NAMESPACE} istio-injection- istio.io/rev=${REVISION} --overwrite

   kubectl apply --context=${CTX_1} -n ${GATEWAY_NAMESPACE} -f ./anthos-service-mesh/samples/gateways/istio-ingressgateway
   ```
   
   Output
   ```
   $ kubectl --context=${CTX_1} -n ${GATEWAY_NAMESPACE} get po,svc
   NAME                                        READY   STATUS    RESTARTS   AGE
   pod/istio-ingressgateway-66d9b945dc-46852   1/1     Running   0          31s
   pod/istio-ingressgateway-66d9b945dc-ftn8z   1/1     Running   0          31s
   pod/istio-ingressgateway-66d9b945dc-kfnsv   1/1     Running   0          31s

   NAME                           TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)                                      AGE
   service/istio-ingressgateway   LoadBalancer   10.24.1.196   34.132.129.229   15021:30640/TCP,80:30051/TCP,443:31968/TCP   35s
   ```
    
- Gateway, VirtualService 정의
   ```
   $ kubectl --context=${CTX_1} apply -f ./kube/asm-nw-ingress.yaml --namespace ${NAMESPACE}
   $ kubectl --context=${CTX_1} --namespace ${NAMESPACE} get gateway,virtualservice
   NAME                                           AGE
   gateway.networking.istio.io/whereami-gateway   49s
   
   NAME                                             GATEWAYS            HOSTS   AGE
   virtualservice.networking.istio.io/whereami-vs   ["whereami-gateway"]   ["*"]   46s
   ```
 
- istio-ingressgateway 의 EXTERNAL-IP(L4 LoadBalancer)로 호출 확인 (34.132.129.229)
 
   ***모든 트래픽은 클러스터 내의 Pod로만 전달됨,단 외부에서도 호출 가능***
   ```
   $ curl 34.132.129.229
   {
     "cluster_name": "asm-multi-neg-1",
     "host_header": "34.132.129.229",
     "pod_name": "whereami-deployment-5755d8b68b-kxzzx",
     "pod_name_emoji": "⏹",
     "project_id": "kwlee-goog-sandbox",
     "timestamp": "2022-04-28T06:22:41",
     "zone": "us-central1-c"
   }
   $ curl 34.132.129.229
   {
     "cluster_name": "asm-multi-neg-1",
     "host_header": "34.132.129.229",
     "pod_name": "whereami-deployment-5755d8b68b-kx4ss",
     "pod_name_emoji": "😅",
     "project_id": "kwlee-goog-sandbox",
     "timestamp": "2022-04-28T06:22:43",
     "zone": "us-central1-c"
   }
   $ curl 34.132.129.229
   {
     "cluster_name": "asm-multi-neg-1",
     "host_header": "34.132.129.229",
     "pod_name": "whereami-deployment-5755d8b68b-kxzzx",
     "pod_name_emoji": "⏹",
     "project_id": "kwlee-goog-sandbox",
     "timestamp": "2022-04-28T06:22:47",
     "zone": "us-central1-c"
   }
   $ curl 34.132.129.229
   {
     "cluster_name": "asm-multi-neg-1",
     "host_header": "34.132.129.229",
     "pod_name": "whereami-deployment-5755d8b68b-kx4ss",
     "pod_name_emoji": "😅",
     "project_id": "kwlee-goog-sandbox",
     "timestamp": "2022-04-28T06:22:51",
     "zone": "us-central1-c"
   }

![image](https://user-images.githubusercontent.com/61114855/165700747-4b4fbc3a-e3ad-41f4-85c0-05f1f4a8e282.png)


## 6. 멀티 클러스터 메시 설정 
- 두 클러스터를 단일 Anthos Service Mesh에 결합 하고 클러스터 간 부하 분산을 사용 설정

- [create firewall rule](https://cloud.google.com/service-mesh/docs/unified-install/gke-install-multi-cluster#create_firewall_rule)   
   ```
   function join_by { local IFS="$1"; shift; echo "$*"; }
   ALL_CLUSTER_CIDRS=$(gcloud container clusters list --project $PROJECT_1 --format='value(clusterIpv4Cidr)' | sort | uniq)
   ALL_CLUSTER_CIDRS=$(join_by , $(echo "${ALL_CLUSTER_CIDRS}"))
   ALL_CLUSTER_NETTAGS=$(gcloud compute instances list --project $PROJECT_1 --format='value(tags.items.[0])' | sort | uniq)
   ALL_CLUSTER_NETTAGS=$(join_by , $(echo "${ALL_CLUSTER_NETTAGS}"))   
   ```
   ```
   gcloud compute firewall-rules create istio-multicluster-pods \
    --allow=tcp,udp,icmp,esp,ah,sctp \
    --direction=INGRESS \
    --priority=900 \
    --source-ranges="${ALL_CLUSTER_CIDRS}" \
    --target-tags="${ALL_CLUSTER_NETTAGS}" --quiet
    ```

- [클러스터간 엔드포인트 검색 구성](https://cloud.google.com/service-mesh/docs/unified-install/gke-install-multi-cluster#configure_endpoint_discovery_between_clusters)
   ```
   ./asmcli create-mesh \
    ${PROJECT_1} \
    ${PROJECT_1}/${LOCATION_1}/${CLUSTER_1} \
    ${PROJECT_2}/${LOCATION_2}/${CLUSTER_2}
   ```   
- 구성 확인
   ```
   $ gcloud container hub memberships list
   NAME: asm-multi-neg-1
   EXTERNAL_ID: f6b53133-1539-4ee9-b4d1-1eff69a3bbf4

   NAME: asm-multi-neg-2
   EXTERNAL_ID: 203cea48-764c-456f-aede-057a12157ead
   `````

## 7. 멀티 클러스터 메시 테스트

   ***모든 트래픽은 단일 메시로 설정한 두 클러스터 ${CLUSTER_1} 괴 ${CLUSTER_2}의 Pod로 전달됨***

   ```
   $ curl http://34.132.129.229/
   {
     "cluster_name": "asm-multi-neg-1",
     "host_header": "34.132.129.229",
     "pod_name": "whereami-deployment-5755d8b68b-kxzzx",
     "pod_name_emoji": "⏹",
     "project_id": "kwlee-goog-sandbox",
     "timestamp": "2022-04-28T06:37:06",
     "zone": "us-central1-c"
   }
   $ curl http://34.132.129.229/
   {
     "cluster_name": "asm-multi-neg-2",
     "host_header": "34.132.129.229",
     "pod_name": "whereami-deployment-764cbfccdb-vlzfg",
     "pod_name_emoji": "🧑🏽✈",
     "project_id": "kwlee-goog-sandbox",
     "timestamp": "2022-04-28T06:37:08",
     "zone": "asia-northeast1-c"
   }
   $ curl http://34.132.129.229/
   {
     "cluster_name": "asm-multi-neg-1",
     "host_header": "34.132.129.229",
     "pod_name": "whereami-deployment-5755d8b68b-kx4ss",
     "pod_name_emoji": "😅",
     "project_id": "kwlee-goog-sandbox",
     "timestamp": "2022-04-28T06:37:10",
     "zone": "us-central1-c"
   }
   $ curl http://34.132.129.229/
   {
     "cluster_name": "asm-multi-neg-2",
     "host_header": "34.132.129.229",
     "pod_name": "whereami-deployment-764cbfccdb-vlzfg",
     "pod_name_emoji": "🧑🏽✈",
     "project_id": "kwlee-goog-sandbox",
     "timestamp": "2022-04-28T06:37:12",
     "zone": "asia-northeast1-c"
   }
   ```

![image](https://user-images.githubusercontent.com/61114855/165701312-8f5e8d8d-cb7e-48dc-aa4a-699f207c9d76.png)






## 8. istio-ingrssgateway 설치, ${CLUSTER-2}에만

- [Install an ingress gateway](https://cloud.google.com/service-mesh/docs/gateways#deploy_gateways)
   ```
   export GATEWAY_NAMESPACE=istio-ingress

   kubectl create namespace ${GATEWAY_NAMESPACE} --context=${CTX_2} 
   kubectl --context=${CTX_2} label namespace ${GATEWAY_NAMESPACE} istio-injection- istio.io/rev=${REVISION} --overwrite

   kubectl apply --context=${CTX_2} -n ${GATEWAY_NAMESPACE} -f ./anthos-service-mesh/samples/gateways/istio-ingressgateway

   ```
   
   Output
   ```
   $ kubectl --context=${CTX_2} -n ${GATEWAY_NAMESPACE} get po,svc
   NAME                                        READY   STATUS    RESTARTS   AGE
   pod/istio-ingressgateway-66d9b945dc-hlw7q   1/1     Running   0          8s
   pod/istio-ingressgateway-66d9b945dc-jlmtg   1/1     Running   0          8s
   pod/istio-ingressgateway-66d9b945dc-lnjxv   1/1     Running   0          8s

   NAME                           TYPE           CLUSTER-IP    EXTERNAL-IP         PORT(S)                                      AGE
   service/istio-ingressgateway   LoadBalancer   10.88.5.210   35.200.122.133      15021:32628/TCP,80:32344/TCP,443:32249/TCP   8s
   ```

- Gateway, VirtualService 정의
   ```
   $ kubectl --context=${CTX_2} apply -f ./kube/asm-nw-ingress.yaml --namespace ${NAMESPACE}
   $ kubectl --context=${CTX_2} --namespace ${NAMESPACE} get gateway,virtualservice
   NAME                                           AGE
   gateway.networking.istio.io/whereami-gateway   49s
   
   NAME                                             GATEWAYS            HOSTS   AGE
   virtualservice.networking.istio.io/whereami-vs   ["whereami-gateway"]   ["*"]   46s
   ```

- istio-ingressgateway 의 EXTERNAL-IP(L4 LoadBalancer)로 호출 확인 (35.200.122.133)
   ```
   $ curl 35.200.122.133
   {
     "cluster_name": "asm-multi-neg-1",
     "host_header": "35.200.122.133",
     "pod_name": "whereami-deployment-5755d8b68b-kx4ss",
     "pod_name_emoji": "😅",
     "project_id": "kwlee-goog-sandbox",
     "timestamp": "2022-04-28T07:39:46",
     "zone": "us-central1-c"
   }
   admin_@cloudshell:~/multi-cluster-with-asm (kwlee-goog-sandbox)$ curl 35.200.122.133
   {
     "cluster_name": "asm-multi-neg-2",
     "host_header": "35.200.122.133",
     "pod_name": "whereami-deployment-764cbfccdb-vlzfg",
     "pod_name_emoji": "🧑🏽✈",
     "project_id": "kwlee-goog-sandbox",
     "timestamp": "2022-04-28T07:39:48",
     "zone": "asia-northeast1-c"
   }
   admin_@cloudshell:~/multi-cluster-with-asm (kwlee-goog-sandbox)$ curl 35.200.122.133
   {
     "cluster_name": "asm-multi-neg-2",
     "host_header": "35.200.122.133",
     "pod_name": "whereami-deployment-764cbfccdb-dw8ct",
     "pod_name_emoji": "🤦🏾",
     "project_id": "kwlee-goog-sandbox",
     "timestamp": "2022-04-28T07:39:48",
     "zone": "asia-northeast1-c"
   }
   admin_@cloudshell:~/multi-cluster-with-asm (kwlee-goog-sandbox)$ curl 35.200.122.133
   {
     "cluster_name": "asm-multi-neg-2",
     "host_header": "35.200.122.133",
     "pod_name": "whereami-deployment-764cbfccdb-dw8ct",
     "pod_name_emoji": "🤦🏾",
     "project_id": "kwlee-goog-sandbox",
     "timestamp": "2022-04-28T07:39:49",
     "zone": "asia-northeast1-c"
   }
   ```

![image](https://user-images.githubusercontent.com/61114855/165702323-03ac1bdb-138e-41f1-9e98-5541a8518ae9.png)









-------------------
Create Ingressgateway as ClusteIP type instead of Load Balancer Type
   ```
   export GATEWAY_NAMESPACE=istio-ingress

   kubectl create namespace ${GATEWAY_NAMESPACE} --context=${CTX_1} 
   kubectl --context=${CTX_1} label namespace ${GATEWAY_NAMESPACE} istio.io/rev=${REVISION} --overwrite

   kubectl apply --context=${CTX_1} -n ${GATEWAY_NAMESPACE} -f ./anthos-service-mesh/samples/gateways/istio-ingressgateway

   # change service of istio-ingressgateway
   kubectl apply --context=${CTX_1} -n ${GATEWAY_NAMESPACE} -f ./kube/istio-ingressgateway/service-for-istio-ingressgateway-1.yaml

   ```

   ```
   export GATEWAY_NAMESPACE=istio-ingress

   kubectl create namespace ${GATEWAY_NAMESPACE} --context=${CTX_2} 
   kubectl --context=${CTX_2} label namespace ${GATEWAY_NAMESPACE} istio.io/rev=${REVISION} --overwrite

   kubectl apply --context=${CTX_2} -n ${GATEWAY_NAMESPACE} -f ./anthos-service-mesh/samples/gateways/istio-ingressgateway
   # change service of istio-ingressgateway
   kubectl apply --context=${CTX_2} -n ${GATEWAY_NAMESPACE} -f ./kube/istio-ingressgateway/service-for-istio-ingressgateway-2.yaml

   ```
   export GKE_NODE_NETWORK_TAGS_1=gke-asm-multi-neg-1-91a85778-node
   export GKE_NODE_NETWORK_TAGS_2=gke-asm-multi-neg-2-f1cf644c-node

   gcloud compute firewall-rules create fw-allow-health-check-and-proxy-for-cluster-1-80 \
   --network=default \
   --action=allow \
   --direction=ingress \
   --target-tags=${GKE_NODE_NETWORK_TAGS_1} \
   --source-ranges=130.211.0.0/22,35.191.0.0/16 \
   --rules=tcp:80

   gcloud compute firewall-rules create fw-allow-health-check-and-proxy-for-cluster-2-80 \
   --network=default \
   --action=allow \
   --direction=ingress \
   --target-tags=${GKE_NODE_NETWORK_TAGS_2} \
   --source-ranges=130.211.0.0/22,35.191.0.0/16 \
   --rules=tcp:80

   gcloud compute firewall-rules create fw-allow-health-check-and-proxy-for-cluster-1-15021 \
   --network=default \
   --action=allow \
   --direction=ingress \
   --target-tags=${GKE_NODE_NETWORK_TAGS_1} \
   --source-ranges=130.211.0.0/22,35.191.0.0/16 \
   --rules=tcp:15021

   gcloud compute firewall-rules create fw-allow-health-check-and-proxy-for-cluster-2-15021 \
   --network=default \
   --action=allow \
   --direction=ingress \
   --target-tags=${GKE_NODE_NETWORK_TAGS_2} \
   --source-ranges=130.211.0.0/22,35.191.0.0/16 \
   --rules=tcp:15021

   gcloud compute firewall-rules create fw-allow-health-check-and-proxy-for-cluster-1-5000 \
   --network=default \
   --action=allow \
   --direction=ingress \
   --target-tags=${GKE_NODE_NETWORK_TAGS_1} \
   --source-ranges=130.211.0.0/22,35.191.0.0/16 \
   --rules=tcp:5000

   gcloud compute firewall-rules create fw-allow-health-check-and-proxy-for-cluster-2-5000 \
   --network=default \
   --action=allow \
   --direction=ingress \
   --target-tags=${GKE_NODE_NETWORK_TAGS_2} \
   --source-ranges=130.211.0.0/22,35.191.0.0/16 \
   --rules=tcp:5000

   gcloud compute firewall-rules create istio-webhook \
   --direction=INGRESS \
   --priority=1000 \
   --network=default \
   --action=ALLOW \
   --source-ranges=0.0.0.0/0 \
   --rules=tcp:15017

   ```

   output
   ```
   $ gcloud compute network-endpoint-groups list
   NAME: asm-ingress-http-1
   LOCATION: us-central1-c
   ENDPOINT_TYPE: GCE_VM_IP_PORT
   SIZE: 3

   NAME: asm-ingress-https-1
   LOCATION: us-central1-c
   ENDPOINT_TYPE: GCE_VM_IP_PORT
   SIZE: 3

   NAME: asm-ingress-http-2
   LOCATION: asia-northeast1-c
   ENDPOINT_TYPE: GCE_VM_IP_PORT
   SIZE: 3

   NAME: asm-ingress-https-2
   LOCATION: asia-northeast1-c
   ENDPOINT_TYPE: GCE_VM_IP_PORT
   SIZE: 3
   ```


Load Balancing --> CREATE LOAD BALANCER --> START CONFIGURATION ("HTTP(S) LOAD BALANCER")
- From Internet to my VMs or serverless services
- Classic HTTP(S) Load Balancer


