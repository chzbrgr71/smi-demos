watch kubectl get pod,svc,deploy,canaries -n test

watch kubectl get trafficsplits podinfo -n test -o yaml

kubectl apply -f ./demo/deployment.yaml
kubectl apply -f ./demo/hpa.yaml
kubectl apply -f ./demo/podinfo-canary.yaml

kubectl -n test set image deployment/podinfo podinfod=quay.io/stefanprodan/podinfo:1.4.4

k port-forward svc/podinfo -n test 9898:9898
