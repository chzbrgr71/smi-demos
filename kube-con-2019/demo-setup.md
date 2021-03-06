### KubeCon North America 2019 Demo

This demo is a part of my talk from KubeCon North America 2019 in San Diego.

https://sched.co/Uadd 

"Supercharge Your Microservices CI/CD with Service Mesh and Kubernetes"
Thursday, Nov 21, 2019, 3:20-3:55pm

In this session, I will dig deep into how you can integrate Service Mesh into deployment pipelines and automate these kinds of CI/CD methods. I will talk about observability using projects such as Prometheus and how it is key to validate candidate releases with real time latency statistics down to specific paths/methods.

https://youtu.be/SMoaem3UBag 

#### Cluster Setup

* Install AKS

* Install Helm

    ```bash
    kubectl apply -f rbac-config.yaml
    helm init --service-account tiller --upgrade
    ```

* Install NGINX Ingress Controller (using Helm)

    ```bash
    kubectl create ns ingress
    kubectl annotate namespace ingress linkerd.io/inject=enabled
    helm install stable/nginx-ingress --namespace=ingress --name=nginx-ingress
    ```

* Install Linkerd (I used stable-2.6.0) https://linkerd.io/2/getting-started 

    ```bash
    curl -sL https://run.linkerd.io/install | sh
    linkerd check --pre
    linkerd install | kubectl apply -f -
    linkerd check
    ```

* Install Flagger

    ```bash
    kubectl apply -k github.com/weaveworks/flagger//kustomize/linkerd
    ```

    Add instructions to enable Slack for Flagger
    https://docs.flagger.app/usage/alerting 

* Flagger Sample (podinfo)

    ```bash
    # setup
    watch kubectl get deploy,pod,hpa,svc,ingress,canary -n test
    linkerd dashboard

    kubectl create ns test
    kubectl annotate namespace test linkerd.io/inject=enabled
    
    kubectl apply -f tester.yaml --namespace test
    kubectl apply -f podinfo.yaml --namespace test
    kubectl apply -f podinfo-ingress.yaml --namespace test

    kubectl apply -f canary.yaml

    # start canary test (stefanprodan/podinfo:3.1.0)
    kubectl -n test set image deployment/podinfo podinfod=stefanprodan/podinfo:3.0.0
    kubectl -n test set image deployment/podinfo podinfod=stefanprodan/podinfo:3.1.0
    kubectl -n test set image deployment/podinfo podinfod=stefanprodan/podinfo:3.1.1
    kubectl -n test set image deployment/podinfo podinfod=stefanprodan/podinfo:3.1.2
    kubectl -n test set image deployment/podinfo podinfod=stefanprodan/podinfo:3.1.3
    kubectl -n test set image deployment/podinfo podinfod=stefanprodan/podinfo:3.1.4

    # pod labels
    podinfo-primary -> app=podinfo-primary
    podinfo (canary) -> app=podinfo

    # service selector
    service/podinfo -> app=podinfo-primary
    service/podinfo-canary -> app=podinfo
    service/podinfo-primary -> app=podinfo-primary
    service/podinfo-external -> app=podinfo

    # load test
    while true; do curl http://podinfo.test.svc:9898/; sleep 1; done
    ```

* Service Tracker Demo

    > Follow steps to setup pre-requisites in Azure as described here. https://github.com/chzbrgr71/kubernetes-hackfest/blob/master/labs/build-application/README.md 

    ```bash
    # install app
    kubectl create ns hackfest
    kubectl annotate namespace hackfest linkerd.io/inject=enabled
    
    kubectl apply -f tester.yaml --namespace hackfest
    kubectl create secret generic cosmos-db-secret --from-literal=user=$MONGODB_USER --from-literal=pwd=$MONGODB_PASSWORD --from-literal=appinsights=$APPINSIGHTS_INSTRUMENTATIONKEY -n hackfest

    kubectl apply -f ./data-api.yaml --namespace hackfest
    kubectl apply -f ./flights-api.yaml --namespace hackfest
    kubectl apply -f ./quakes-api.yaml --namespace hackfest
    kubectl apply -f ./weather-api.yaml --namespace hackfest
    kubectl apply -f ./service-tracker-ui.yaml --namespace hackfest
    kubectl apply -f ./ingress.yaml --namespace hackfest

    # initialize canary
    kubectl apply -f canary.yaml

    # setup demo
    deploy ACIs
    Github pages with branch https://github.com/chzbrgr71/kubernetes-hackfest 
    watch kubectl get deploy,pod,service,canary,trafficsplit -n hackfest
    linkerd dashboard
    http://flights-api.brianredmond.io/status 
    http://23.99.133.108:8080 
    kubectl exec -it flagger-loadtester-8649c9d49f-4bzjw -n hackfest bash
    while true; do curl http://flights-api.hackfest.svc:3003/status; echo $'\n'; sleep 1; done

    # run upgrades with canary test
    kubectl -n hackfest set image deployment/flights-api flights-api=briaracr.azurecr.io/hackfest/flights-api:1.1.6

    kubectl -n hackfest set image deployment/flights-api flights-api=briaracr.azurecr.io/hackfest/flights-api:1.1.95-error

    kubectl describe canary flights-api -n hackfest

    # test
    kubectl exec -it flagger-loadtester-8649c9d49f-kk5nx -n hackfest bash
    while true; do curl http://flights-api.hackfest.svc:3003/status; echo $'\n'; sleep 1; done
    curl -s http://flights-api-canary.hackfest:3003/status | grep payload
    ```

* Reset demo

    ```bash
    kubectl delete canary flights-api -n hackfest
    kubectl delete deploy flights-api -n hackfest
    kubectl apply -f ./flights-api.yaml --namespace hackfest
    ```

* Build new container image - Manually

    ```bash
    export ACRNAME=briaracr
    export IMAGE_TAG=1.2.0

    az acr build -t hackfest/flights-api:$IMAGE_TAG -r $ACRNAME --no-logs ~/source/kubernetes-hackfest/app/flights-api

    az acr build -t hackfest/quakes-api:$IMAGE_TAG -r $ACRNAME --no-logs ~/source/kubernetes-hackfest/app/quakes-api

    az acr build -t hackfest/weather-api:$IMAGE_TAG -r $ACRNAME --no-logs ~/source/kubernetes-hackfest/app/weather-api
    ```

* Load Tester ACI pods

    ```bash
    # run all
    for i in 1 2; do
        az container create --name quakes-load-test${i} -l southcentralus --image chzbrgr71/loadtest:v2.0 --resource-group kubecon-2019 -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=quakes-api.brianredmond.io/latest
        az container create --name weather-load-test${i} -l southcentralus --image chzbrgr71/loadtest:v2.0 --resource-group kubecon-2019 -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=weather-api.brianredmond.io/latest
    done

    # backup cluster
    for i in 1 2; do
        az container create --name quakes-load-test${i} -l southcentralus --image chzbrgr71/loadtest:v2.0 --resource-group kubecon-2019 -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=23.99.203.105:3012/latest
        az container create --name weather-load-test${i} -l southcentralus --image chzbrgr71/loadtest:v2.0 --resource-group kubecon-2019 -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=52.173.206.133:3015/latest
    done

    # delete all
    for i in 1 2; do
        az container delete --yes --resource-group kubecon-2019 --name quakes-load-test${i}
        az container delete --yes --resource-group kubecon-2019 --name weather-load-test${i}
    done    
    ```

#### Resources

* https://linkerd.io/2/getting-started 
* https://docs.flagger.app/usage/linkerd-progressive-delivery 
* https://github.com/stefanprodan/podinfo
* https://github.com/chzbrgr71/podinfo 



