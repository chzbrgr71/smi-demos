### Service Mesh Demos


#### Cluster Setup

* Install AKS

    ```bash
    export RGNAME=service-mesh
    export LOCATION=eastus2
    export CLUSTERNAME=briar-aks-osm
    export K8SVERSION=1.19.7
    export VMSIZE=Standard_D2_v2
    export NODECOUNT=5
    export AZUREMONITOR=/subscriptions/471d33fd-a776-405b-947c-467c291dc741/resourcegroups/monitoring/providers/microsoft.operationalinsights/workspaces/briar-aks-monitoring

    az group create --name $RGNAME --location $LOCATION

    az aks create \
    --resource-group $RGNAME \
    --name $CLUSTERNAME \
    --node-count $NODECOUNT \
    --kubernetes-version $K8SVERSION \
    --location $LOCATION \
    --vm-set-type VirtualMachineScaleSets \
    --enable-managed-identity \
    --node-osdisk-type Ephemeral \
    --node-osdisk-size 30 \
    --network-plugin azure \
    --workspace-resource-id $AZUREMONITOR \
    --enable-addons monitoring,open-service-mesh \
    --no-wait

    az aks get-credentials -n $CLUSTERNAME -g $RGNAME

    az aks list -g $RGNAME -o json | jq -r '.[].addonProfiles.openServiceMesh.enabled'

    PROM_POD_NAME=$(kubectl get pods -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
    kubectl --namespace default port-forward $PROM_POD_NAME 9090
    ```

### Install Service Mesh

* Linkerd

    ```bash
    curl -sL https://run.linkerd.io/install | sh
    linkerd check --pre
    linkerd install | kubectl apply -f -
    linkerd check
    kubectl get pod -n linkerd
    linkerd dashboard &
    ```

* Open Service Mesh

    ```bash
    osm install \
      --enable-permissive-traffic-policy \
      --deploy-grafana \
      --deploy-jaeger \
      --deploy-prometheus \
      --enable-egress

    for i in bookstore bookbuyer bookthief bookwarehouse; do kubectl delete ns $i; done
    ```

* TCP Route for Mongo

https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-access/v1alpha3/traffic-access.md#example-implementation-for-l7 

### Demo

* Install NGINX Ingress Controller (using Helm)

    ```bash
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update

    kubectl create ns ingress
    kubectl annotate namespace ingress linkerd.io/inject=enabled
    helm install nginx-ingress ingress-nginx/ingress-nginx --namespace=ingress
    ```

* App

    ```bash
    # install app
    kubectl create ns tracker

    # linkerd
    kubectl annotate namespace tracker linkerd.io/inject=enabled

    # osm
    kubectl label namespace tracker openservicemesh.io/monitored-by=osm
    kubectl annotate namespace tracker openservicemesh.io/sidecar-injection=enabled

    kubectl label namespace nginx openservicemesh.io/monitored-by=osm
    kubectl annotate namespace nginx openservicemesh.io/sidecar-injection=enabled

    kubectl apply -f ./service-tracker/mongodb.yaml --namespace tracker
    kubectl apply -f ./service-tracker/data-api.yaml --namespace tracker
    kubectl apply -f ./service-tracker/flights-api.yaml --namespace tracker
    kubectl apply -f ./service-tracker/quakes-api.yaml --namespace tracker
    kubectl apply -f ./service-tracker/weather-api.yaml --namespace tracker
    kubectl apply -f ./service-tracker/service-tracker-ui.yaml --namespace tracker

    # SMI
    kubectl apply -f ./service-tracker/data-api-new.yaml --namespace tracker
    kubectl apply -f traffic-split.yaml --namespace tracker
    ```

* Load Test

    ```bash
    # bash
    while true; do curl http://52.154.63.0:3012/latest; echo $'\n'; sleep 1; done

    az container create --name quakes-load-test2 -l centralus --image chzbrgr71/loadtest:v3.1 --resource-group service-mesh -o tsv --cpu 1 --memory 1 --environment-variables LOAD_DURATION=-1 LOAD_RATE=2 URL=http://52.154.63.0:3012/latest
    az container delete --yes --resource-group service-mesh --name quakes-load-test2

    # ACI
    for i in 1 2 3; do
        az container create --name quakes-load-test${i} -l centralus --image chzbrgr71/loadtest:v3.1 --resource-group service-mesh -o tsv --cpu 1 --memory 1 --environment-variables LOAD_DURATION=-1 LOAD_RATE=2 URL=http://52.154.63.0:3012/latest
        az container create --name weather-load-test${i} -l centralus --image chzbrgr71/loadtest:v3.1 --resource-group service-mesh -o tsv --cpu 1 --memory 1 --environment-variables LOAD_DURATION=-1 LOAD_RATE=2 URL=http://52.154.161.94:3015/latest
        az container create --name flights-load-test${i} -l centralus --image chzbrgr71/loadtest:v3.1 --resource-group service-mesh -o tsv --cpu 1 --memory 1 --environment-variables LOAD_DURATION=-1 LOAD_RATE=2 URL=http://52.154.61.51:3003/latest
    done

    # ACI delete
    for i in 1 2 3; do
        az container delete --yes --resource-group service-mesh --name quakes-load-test${i}
        az container delete --yes --resource-group service-mesh --name weather-load-test${i}
        az container delete --yes --resource-group service-mesh --name flights-load-test${i}
    done 
    ```


