# Setup new CIVO cluster

*NOTE* This document assumes that you are logged into the CIVO cli and running this from the root of the project

## Pre-Requisietes

1. Create the cluster from the CIVO UI and take note of the IP address
1. Download and setup the new KubeConfig file
1. Add new cluster IP to DigitalOcean Databse
1. Add new cluster IP to AWS Route 53 for the appropriate DNs below
    - DR Cluster
        - `ldkube.com`
    - PROD Cluster
        - `ldkube.io`
        - `karenandlucas.com`
        - `countmybreaths.com`

## Namespaces and Secrets

Create namespaces
```
$ kubectl apply -f namespaces/
```

Create secrets (this step requires access to the right files)
```
$ kubectl apply -f .secrets/
```


## Cert Manager

Get the new CIVO cluster name
```
$ civo kubernetes ls 

+--------------------------------------+----------------+--------+-------+-------+--------+
| ID                                   | Name           | Region | Nodes | Pools | Status |
+--------------------------------------+----------------+--------+-------+-------+--------+
| sdfsdfdf-ffsd-fdfd-dfs-23fdsfsdsdsfs | my-cluster-01  | NYC1   |     3 |     1 | ACTIVE |
+--------------------------------------+----------------+--------+-------+-------+--------+
```

Install Cert Manager 
```
$ civo kubernetes applications add cert-manager -c my-cluster-01
```

Install the Cert Issuer
```
$ kubectl apply -f civo/issuer.yml
```

## Install nginx

Apply the Helm Chart
```
$ helm install nginx nginx-stable/nginx-ingress
```

Test by applying the example manifest file included in this repo
```
$ kubectl apply -f civo/echo.yml
```

Watch the echo cert until it shows as ready
```
$ kubectl get cert -n echo -w 

NAME       READY   SECRET     AGE
echo-tls   True    echo-tls   21h
```

Curl the echo url to test the ingress is working properly
```
$ curl echo.ldkube.com

echo
```

## Install argo
```
$ civo kubernetes applications add argo-cd -c my-cluster-01
```

Confirm installation 
```
$ civo kubernetes applications show argo-cd my-cluster-01

0.1 Argo CD

0.1.1 Access Argo CD UI

... 
```

Get the initial admin password
```
$ kubectl -n argocd get secret argocd-initial-admin-secret -o yaml 

apiVersion: v1
data:
  password: cGFzc3dvcmQK
kind: Secret

...
```

Decode the password from above
```
$ echo cGFzc3dvcmQK | base64 -d

password
```


Find the argo pod name
```
$ kubectl get pod -n argocd | grep argo-cd-argocd-server

argo-cd-argocd-server-12vsdf-12dg                      1/1     Running   0             22h
```

Port forward the server and login at http://localhost:8080 using the user 'admin' and password decoded above

```
$ kubectl port-forward pod/argo-cd-argocd-server-12vsdf-12dg 8080:8080 -n argocd  

Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

## Datadog

Generate an apikey for the new cluster using the datadog ui

Install the Datadog Helm Chart
```
$ helm upgrade datadog -f datadog-values-<CLUSTER_NAME>.yaml --set datadog.site='us5.datadoghq.com' --set datadog.apiKey=<API_KEY> datadog/datadog --install
```

## Apps

Install the argo apps
```
$ kubectl apply -f apps/
```

## Cutover to backup cluster

Get the cluster IP
```
$ civo kubernetes show my-cluster | grep "External IP" 

           External IP : 555.5.555.555
```

Update the below AWS Route 53 DNs with the new cluster IP

- `ldkube.io`
- `karenandlucas.com`
- `countmybreaths.com`

Apply the prod ingress manifests to the new cluster
```
$ kubectl apply -f ingresses/prod/
```


