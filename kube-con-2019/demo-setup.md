### KubeCon North America 2019 Demo

This demo is a part of my talk from KubeCon North America 2019 in San Diego.

"Supercharge Your Microservices CI/CD with Service Mesh and Kubernetes"

In this session, I will dig deep into how you can integrate Service Mesh into deployment pipelines and automate these kinds of CI/CD methods. I will talk about observability using projects such as Prometheus and how it is key to validate candidate releases with real time latency statistics down to specific paths/methods.


#### Cluster Setup

* Install AKS

* Install Helm

* Install NGINX Ingress Controller (using Helm)

    ```bash
    helm install stable/nginx-ingress --namespace=kube-system --name=nginx-ingress
    ```

* Install Linkerd (I used stable-2.6.0) https://linkerd.io/2/getting-started 

    ```bash
    linkerd install | kubectl apply -f -
    ```

* Install Flagger

    ```bash
    kubectl apply -k github.com/weaveworks/flagger//kustomize/linkerd
    ```




#### Resources

* https://linkerd.io/2/getting-started 
* https://docs.flagger.app/usage/linkerd-progressive-delivery 



