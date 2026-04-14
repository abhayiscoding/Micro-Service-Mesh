# Microservice Mesh using Istio
- 3 services, covering service-to-service authentication (mTLS) and traffic splitting (canary deployments)

## Start
Install Istio, prefer istioctl over helm
> kind create cluster --config=cluster.yaml --name=cluster1
> curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.28.0 sh -
// donwnload istio zip file, extract it if curl is not working(in my case)
> cd istio-1.28.0   
// add path to env variable
> export PATH=$PWD/bin:$PATH
> istioctl install --set profile=default -y
> istioctl version
> kubectl create namespace mesh-demo
> kubectl label namespace mesh-demo istio-injection=enabled
> kubectl get pods -n istio-system  // verify installation

After creating service-a,b,c yaml files, run
> kubectl apply -f service-a.yaml
> kubectl apply -f service-b.yaml
> kubectl apply -f service-c.yaml
Create peer-auth.yaml
> kubectl apply -f peer-auth.yaml
<!-- Now all traffic inside the mesh must be mTLS encrypted. -->

<!-- Created destination-rule.yaml for Traffic Splitting (90% v1, 10% v2) -->
Created virtual-service.yaml, 90/10 split
> kubectl apply -f destination-rule.yaml
> kubectl apply -f virtual-service.yaml

Verification:
<!-- Checking mTLS is working -->
> kubectl logs -n mesh-demo deployment/service-a istio-proxy | grep "TLS"
    // controlPlaneAuthPolicy: MUTUAL_TLS

<!-- Checking traffic splitting by Exec into service-a and curl repeatedly -->
> kubectl exec -n mesh-demo deployment/service-a -- sh -c "for i in \$(seq 1 20); do curl -s http://service-b:8080/; done"

Service-c is only reachable with mTLS, From service-a   // should reuturn response from service-c
> kubectl exec -n mesh-demo deployment/service-a -- curl -s http://service-c:8080/


In case to editing and reapplying .yaml files, restart pods
> kubectl rollout restart deployment -n mesh-demo

// Add kiali dashboard
> istioctl install --set profile=demo -y
> kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/addons/kiali.yaml
// Prometheus addon
> kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/addons/prometheus.yaml
> kubectl get pods -n istio-system | grep prometheus
// grafana addon
> kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/addons/grafana.yaml
> kubectl rollout restart deployment kiali -n istio-system  // restart kiali
> istioctl dashboard kiali
// [localhost:20001](http://localhost:20001/kiali/console/overview?duration=60&refresh=60000)


## CleanUp
> kubectl delete namespace mesh-demo
> kubectl delete namespace istio-system
> istioctl uninstall --purge -y
> kubectl delete crd $(kubectl get crd | grep 'istio.io' | awk '{print $1}')
Also remove its path from environment variables
> Kind delete cluster --name=cluster1
