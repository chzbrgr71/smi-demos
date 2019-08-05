# Open Source Summit 2019 - San Diego

## Lightning Talk: Service Mesh Up and Running in 5 Minutes

#### Pre-demo

* Run app

```bash
kubectl apply -f ./k8s/ -n tracker
```

* Generate activity (Artillery)

https://github.com/chzbrgr71/load-test-artillery 

```bash
docker run -d --name load-test1 -e "load_duration=-1" -e "load_rate=1" -e "load_url=104.40.29.56:3003/latest" chzbrgr71/loadtest:v2.0

az container create --name flights-load-test1 --image chzbrgr71/loadtest:v2.0 --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=5 load_url=104.40.29.56:3003/latest

az container delete --yes --resource-group aci --name flights-load-test1

# run all
export FLIGHTS_IP=104.40.29.56
export QUAKES_IP=104.40.22.130
export WEATHER_IP=104.40.25.113

for i in 1 2 3; do
    az container create --name flights-load-test${i} -l westus --image chzbrgr71/loadtest:v2.0 --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=$FLIGHTS_IP:3003/latest
    az container create --name quakes-load-test${i} -l westus --image chzbrgr71/loadtest:v2.0 --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=$QUAKES_IP:3012/latest
    az container create --name weather-load-test${i} -l westus --image chzbrgr71/loadtest:v2.0 --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=$WEATHER_IP:3015/latest
done

for i in 1 2 3; do
    az container delete --yes --resource-group aci --name flights-load-test${i}
    az container delete --yes --resource-group aci --name quakes-load-test${i}
    az container delete --yes --resource-group aci --name weather-load-test${i}
done
```


#### Demo steps

* Show application

* Install Linkerd

```bash
linkerd check --pre

linkerd install | kubectl apply -f -

kubectl -n linkerd get deploy

linkerd dashboard
```

* Mesh application

```
linkerd inject ./manifests/data-api.yaml | kubectl apply -n tracker -f -
linkerd inject ./manifests/flights-api.yaml | kubectl apply -n tracker -f -
linkerd inject ./manifests/quakes-api.yaml | kubectl apply -n tracker -f -
linkerd inject ./manifests/weather-api.yaml | kubectl apply -n tracker -f -
linkerd inject ./manifests/service-tracker-ui.yaml | kubectl apply -n tracker -f -
```

* Show Dashboard activity


* SMI Traffic Split

