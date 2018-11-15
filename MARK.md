# Prometheus operator

To download :

   git clone https://github.com/coreos/prometheus-operator

## Docs with instructions below 
[https://github.com/coreos/prometheus-operator/tree/master/contrib/kube-prometheus]

## Install
    git clone https://github.com/coreos/prometheus-operator
    cd /tree/master/contrib/kube-prometheus
    kubectl create -f manifests/ || true
    kubectl create -f manifests/ 2>/dev/null || true

## Access
### Prometheus

    kubectl --namespace monitoring port-forward svc/prometheus-k8s 9090
Then access via http://localhost:9090

### Grafana

    kubectl --namespace monitoring port-forward svc/grafana 3000
Then access via http://localhost:3000 
Use the default grafana user:password of admin:admin

# Traefik

   git clone https://github.com/containous/traefik.git

kubernetes examples :
https://github.com/containous/traefik/tree/master/examples/k8s

https://github.com/Zenika/traefik-gke-demo

https://estl.tech/configuring-https-to-a-web-service-on-google-kubernetes-engine-2d71849520d
