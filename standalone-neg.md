## 1. 최초 변수 설정
    ```
    # CLUSTER-1 for standalone-neg
    export PROJECT_1=kw-gke-prj
    export CLUSTER_1=standalone-neg-1
    export LOCATION_1=us-central1-c
    
    # CLUSTER-2 for standalone-neg
    export PROJECT_2=kw-gke-prj
    export CLUSTER_2=standalone-neg-2
    export LOCATION_2=asia-northeast1-c
    ```
## 2. GKE 클러스터 생성 및 인증정보 설정
    ```
    gcloud container clusters create ${CLUSTER_1} \
        --project=${PROJECT_1} \
        --zone=${LOCATION_1} \
        --machine-type=e2-standard-4 \
        --num-nodes=2 \
        --workload-pool=${PROJECT_1}.svc.id.goog
    
    gcloud container clusters create ${CLUSTER_2} \
        --project=${PROJECT_2} \
        --zone=${LOCATION_2} \
        --machine-type=e2-standard-4 \
        --num-nodes=2 \
        --workload-pool=${PROJECT_2}.svc.id.goog

    gcloud container clusters get-credentials ${CLUSTER_1} \
    --project=${PROJECT_1} \
    --zone=${LOCATION_1}

    gcloud container clusters get-credentials ${CLUSTER_2} \
    --project=${PROJECT_2} \
    --zone=${LOCATION_2}
    
    export CTX_1="gke_${PROJECT_1}_${LOCATION_1}_${CLUSTER_1}"
    export CTX_2="gke_${PROJECT_2}_${LOCATION_2}_${CLUSTER_2}"
    ```

    ```
    ```







