# Open Source Summit 2019 - San Diego

## Lightning Talk: Service Mesh Up and Running in 5 Minutes

#### Pre-demo

* Kubernetes secret

```bash
export RGNAME=oss-summit-west
export COSMOSNAME=briarosssummit
export APPINSIGHTS_INSTRUMENTATIONKEY=<replace me>
export MONGODB_USER=$(az cosmosdb show --name $COSMOSNAME --resource-group $RGNAME --query "name" -o tsv)
export MONGODB_PASSWORD=$(az cosmosdb list-keys --name $COSMOSNAME --resource-group $RGNAME --query "primaryMasterKey" -o tsv)

kubectl create secret generic cosmos-db-secret --from-literal=user=$MONGODB_USER --from-literal=pwd=$MONGODB_PASSWORD --from-literal=appinsights=$APPINSIGHTS_INSTRUMENTATIONKEY -n tracker
```

* Run app

```bash
# get data-api running first
kubectl apply -f ./k8s/data-api.yaml -n tracker

kubectl apply -f ./k8s/ -n tracker
```

* Generate activity (Artillery)

https://github.com/chzbrgr71/load-test-artillery 

```bash
# simple test
az container create --name flights-load-test1 --image chzbrgr71/loadtest:v2.0 --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=5 load_url=104.40.29.56:3003/latest

az container delete --yes --resource-group aci --name flights-load-test1

# run all
export FLIGHTS_IP=$(kubectl get svc --namespace tracker flights-api -o jsonpath='{.status.loadBalancer.ingress[0].ip}') && echo $FLIGHTS_IP
export QUAKES_IP=$(kubectl get svc --namespace tracker quakes-api -o jsonpath='{.status.loadBalancer.ingress[0].ip}') && echo $QUAKES_IP
export WEATHER_IP=$(kubectl get svc --namespace tracker weather-api -o jsonpath='{.status.loadBalancer.ingress[0].ip}') && echo $WEATHER_IP

for i in 1 2 3; do
    az container create --name flights-load-test${i} -l westus --image chzbrgr71/loadtest:v2.0 --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=$FLIGHTS_IP:3003/latest
    az container create --name quakes-load-test${i} -l westus --image chzbrgr71/loadtest:v2.0 --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=$QUAKES_IP:3012/latest
    az container create --name weather-load-test${i} -l westus --image chzbrgr71/loadtest:v2.0 --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=$WEATHER_IP:3015/latest
done
```


#### Live Demo Steps

* Show application

```bash
export WEB_URL=http://$(kubectl get svc --namespace tracker service-tracker-ui -o jsonpath='{.status.loadBalancer.ingress[0].ip}'):8080 && echo $WEB_URL
```

* Install Linkerd

```bash
linkerd check --pre

linkerd install | kubectl apply -f -

kubectl -n linkerd get deploy

linkerd dashboard

linkerd stat namespaces
```

* Mesh application

```bash
# data-api first
linkerd inject ./k8s/data-api.yaml | kubectl apply -n tracker -f -

# the rest
linkerd inject ./k8s/ | kubectl apply -n tracker -f -

# 1 by 1
linkerd inject ./k8s/data-api.yaml | kubectl apply -n tracker -f -
linkerd inject ./k8s/flights-api.yaml | kubectl apply -n tracker -f -
linkerd inject ./k8s/quakes-api.yaml | kubectl apply -n tracker -f -
linkerd inject ./k8s/weather-api.yaml | kubectl apply -n tracker -f -
linkerd inject ./k8s/service-tracker-ui.yaml | kubectl apply -n tracker -f -
```

* Show Dashboard

```bash
linkerd dashboard
```

* Interesting Linkerd commands

```bash
linkerd stat namespaces

linkerd stat deployments -n tracker

linkerd top namespace/tracker
```

* SMI Traffic Split

    * Deploy canary build
    
    ```bash
    kubectl apply -f ./k8s/flights-api-canary.yaml -n tracker
    ```

    * Show SMI YAML

    * Deploy and validate


#### Clean-up

* Stop load testing

```bash
for i in 1 2 3; do
    az container delete --yes --resource-group aci --name flights-load-test${i}
    az container delete --yes --resource-group aci --name quakes-load-test${i}
    az container delete --yes --resource-group aci --name weather-load-test${i}
done
```

* Unmesh app

```bash
linkerd inject ./k8s/ | kubectl delete -n tracker -f -
```

* Uninstall Linkerd

```bash
linkerd install --ignore-cluster | kubectl delete -f -
```