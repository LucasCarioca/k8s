
https://www.civo.com/learn/managing-external-load-balancers-on-civo

install cert manager
```
civo kubernetes applications add cert-manager -c <CLUSTER_NAME>
kubectl apply -f issuer.yml
```

install nginx
```
helm install nginx nginx-stable/nginx-ingress
# to test apply the echo
kubectl apply -f echo.yml
```

install argo
```
civo kubernetes applications add argo-cd -c <CLUSTER_NAME>
kubectl -n argocd get secret argocd-initial-admin-secret -o yaml 
echo <PASSWORD> | base64 -d
kubectl port-forward pod/<ARGO_SERVER_POD_NAME> 8080:8080 -n argocd     
```



install datadog 
```
helm upgrade datadog -f datadog-values.yaml --set datadog.site='us5.datadoghq.com' --set datadog.apiKey=<API_KEY> datadog/datadog --install
```