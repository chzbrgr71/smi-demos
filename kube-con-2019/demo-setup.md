### KubeCon North America 2019 Demo

This demo is a part of my talk from KubeCon North America 2019 in San Diego.

https://sched.co/Uadd 

"Supercharge Your Microservices CI/CD with Service Mesh and Kubernetes"
Thursday, Nov 21, 2019, 3:20-3:55pm

In this session, I will dig deep into how you can integrate Service Mesh into deployment pipelines and automate these kinds of CI/CD methods. I will talk about observability using projects such as Prometheus and how it is key to validate candidate releases with real time latency statistics down to specific paths/methods.


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
    linkerd install | kubectl apply -f -
    ```

* Install Flagger

    ```bash
    kubectl apply -k github.com/weaveworks/flagger//kustomize/linkerd
    ```

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

    ```bash
    # install app
    kubectl annotate namespace hackfest linkerd.io/inject=enabled
    
    kubectl apply -f tester.yaml --namespace hackfest

    kubectl apply -f ./data-api.yaml --namespace hackfest
    kubectl apply -f ./flights-api.yaml --namespace hackfest
    kubectl apply -f ./quakes-api.yaml --namespace hackfest
    kubectl apply -f ./weather-api.yaml --namespace hackfest
    kubectl apply -f ./service-tracker-ui.yaml --namespace hackfest
    kubectl apply -f ./ingress.yaml --namespace hackfest

    # initialize canary
    kubectl apply -f canary.yaml

    # setup demo
    watch kubectl get deploy,pod,service,canary,trafficsplit -n hackfest
    linkerd dashboard
    http://flights-api.brianredmond.io/status 

    # run upgrades with canary test
    kubectl -n hackfest set image deployment/flights-api flights-api=briaracr.azurecr.io/hackfest/flights-api:1.1.6

    kubectl -n hackfest set image deployment/flights-api flights-api=briaracr.azurecr.io/hackfest/flights-api:1.1.7

    kubectl -n hackfest set image deployment/flights-api flights-api=briaracr.azurecr.io/hackfest/flights-api:1.1.8

    kubectl -n hackfest set image deployment/flights-api flights-api=briaracr.azurecr.io/hackfest/flights-api:1.1.95-error

    kubectl describe canary flights-api -n hackfest

    # load test
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
        #az container create --name flights-load-test${i} -l centralus --image chzbrgr71/loadtest:v2.0 --resource-group kubecon-2019 -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=flights-api.brianredmond.io/latest
        az container create --name quakes-load-test${i} -l centralus --image chzbrgr71/loadtest:v2.0 --resource-group kubecon-2019 -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=quakes-api.brianredmond.io/latest
        az container create --name weather-load-test${i} -l centralus --image chzbrgr71/loadtest:v2.0 --resource-group kubecon-2019 -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=weather-api.brianredmond.io/latest
    done

    # delete all
    for i in 1 2; do
        #az container delete --yes --resource-group kubecon-2019 --name flights-load-test${i}
        az container delete --yes --resource-group kubecon-2019 --name quakes-load-test${i}
        az container delete --yes --resource-group kubecon-2019 --name weather-load-test${i}
    done    
    ```

#### Resources

* https://linkerd.io/2/getting-started 
* https://docs.flagger.app/usage/linkerd-progressive-delivery 
* https://github.com/stefanprodan/podinfo
* https://github.com/chzbrgr71/podinfo 



