# Istio Meetup

## Prereqs
Any native k8s cluster.
## Install Traefik
kubectl apply -f https://raw.githubusercontent.com/Conoa/istio/master/deploy-traefik.yaml


## Install Istio
```
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.7.3 TARGET_ARCH=x86_64 sh -
#curl -L https://istio.io/downloadIstio | sh -
cd istio-1.7.3
export PATH=$PATH:$PWD/bin
istioctl install --set profile=demo
kubectl get svc -n istio-system
```

# Install bookinfo application
```
kubectl create ns bookinfo
```

# Clone git (behövs ej för samples)
```
git clone https://github.com/istio/istio.git
```

# Deploy bookinfo in namespace bookinfo
```
kubectl -n bookinfo apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

# Check the deployment in the namespace
Here you can see that it is one container per pod as expected
```
kubectl -n bookinfo get all 
```

# Now remove the deployed application by removing the complete namespace
```
kubectl delete namespace bookinfo
# or
kubectl -n bookinfo delete -f samples/bookinfo/platform/kube/bookinfo.yaml
```

# Second lets deploy the bookinfo microservices with istio injected
Install bookinfo application
```
kubectl create ns bookinfo
```

# Label namespace for automatic injection
```
kubectl label namespace bookinfo istio-injection=enabled
```

# Deploy bookinfo in namespace bookinfo
```
kubectl -n bookinfo apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```
# Check the deployment in the namespace
Here you can see that there is two containers per pod as expected
```
kubectl -n bookinfo get all 
kubectl -n bookinfo get pods
```

# Verify that application works:
This could be nominated as the longest oneliner in the world, looks complicated but it is really not
Get the name of the pod and execute a curl command inside the container running in the pod

It should say: <title>Simple Bookstore App</title>
```
kubectl -n bookinfo exec -it $(kubectl -n bookinfo get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
```

# Deploy the ingress gateway for the application
```
kubectl -n bookinfo apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

# Ensure there is no issues with installation
<PRE>
istioctl analyze
</PRE>

# Steps for constructing the URL for the Ingress Gateway
In my enviroment I will use NodePorts since I have no external LoadBalancer
```
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
# export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
```

# Internal host IP 
```
# export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
```

# External host IP
Lets just use the k8s master public IP for now
```
export INGRESS_HOST=$(curl -s ifconfig.io)
```

# Construct the URL
```
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

# Confirm that the app is accessable from outside the cluster
```
curl -s http://${GATEWAY_URL}/productpage | grep -o "<title>.*</title>"
```

# Try with browser:
```
echo $GATEWAY_URL/productpage
```
# Apply default destination rules (no TLS)
```
kubectl -n bookinfo apply -f samples/bookinfo/networking/destination-rule-all.yaml
```

# Check destination rules
```
kubectl -n bookinfo get destinationrules -o yaml
```

### Bookinfo application is now deployed and ready to be used for Istio Example Scenarios ####


# Check with:
kubectl -n bookinfo get virtualservices   
kubectl -n bookinfo get destinationrules  
kubectl -n bookinfo get gateway           
kubectl -n bookinfo get pods            


## Uninstall
```
kubectl delete namespace bookinfo
# or
istioctl manifest generate --set profile=demo | kubectl delete -f -
```


## Request Routing
### Apply a virtual service
Run the following command to apply the virtual services:
Route all traffic to v1
```
  kubectl apply -n bookinfo -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```
What did we actually do?
```
kubectl -n bookinfo get virtualservices -o yaml
kubectl -n bookinfo get destinationrules -o yaml
```

### Verify routing configuration

Now test with your browser to:

http://$GATEWAY_URL/productpage

The ratings should show no stars as we are routing to v1 which has no stars implemented

## Route based on user identity

Next, you will change the route configuration so that all traffic from a specific user is routed to a specific service version. In this case, all traffic from a user named Jason will be routed to the service reviews:v2.

```
kubectl -n bookinfo apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```
### Verify routing configuration

Now login as jason and go to the same page:

http://$GATEWAY_URL/productpage

The ratings should show have stars as we are routing to v2 which has stars implemented

## Cleanup
```
kubectl -n bookinfo delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```
# Traffic Shifting
Start by routing everything to v1
```
kubectl -n bookinfo apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

Now transfer 50% of the traffic to v3
```
kubectl -n bookinfo apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
```

Now let 100% of the traffic goto v3
```
  kubectl -n bookinfo apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml
```

# Clean up
```
    kubectl -n bookinfo delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```


# Visibility
## Create a secret
Create a secret in your Istio namespace with the credentials that you use to authenticate to Kiali.
First, define the credentials you want to use as the Kiali username and passphrase:

```
export KIALI_USERNAME=$(read -p 'Kiali Username: ' uval && echo -n $uval | base64)
export KIALI_PASSPHRASE=$(read -sp 'Kiali Passphrase: ' pval && echo -n $pval | base64)
```

To create a secret, run the following commands:

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: kiali
  namespace: istio-system
  labels:
    app: kiali
type: Opaque
data:   
  username: $KIALI_USERNAME
  passphrase: $KIALI_PASSPHRASE
EOF
```
## Enable kiali (& Grafana)
```
# istioctl manifest apply --set values.kiali.enabled=true --set values.grafana.enabled=true   # Old 1.5 style
kubectl apply -f samples/addons # Twice due to timing errors

```

## Install demo ingress for kiali and grafana
(Kenneth, kom ihåg att använda DINA yaml filer... Mvh Kenneth)
```
cat <<EOF | kubectl -n istio-system apply -f -
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: kiali-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: 161.47.228.35.bc.googleusercontent.com
    http:
      paths:
      - backend:
          serviceName: kiali
          servicePort: 20001
        path: /kiali
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: grafana-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: 161.47.228.35.bc.googleusercontent.com
    http:
      paths:
      - backend:
          serviceName: grafana
          servicePort: 3000
        path: /grafana
EOF
```

## Generating a service graph

To verify the service is running in your cluster, run the following command:
```
kubectl -n istio-system get svc kiali 
```

## Generate some load
```
watch -n 1 curl -o /dev/null -s -w %{http_code} $GATEWAY_URL/productpage
```

# Grafana doesn't like subpaths...
```
kubectl -n istio-system patch deployment grafana -p '{"spec":{"template":{"spec":{"containers":[{"name":"grafana","env":[{"name":"GF_SERVER_ROOT_URL","value":"/grafana/"},{"name":"GF_SERVER_SERVE_FROM_SUB_PATH","value":"True"}]}]}}}}'
```

# Test Kiali & Grafan
http://177.150.228.35.bc.googleusercontent.com/kiali

http://177.150.228.35.bc.googleusercontent.com/grafana/dashboard/db/istio-mesh-dashboard


# Cleanup Kiali

```
kubectl delete all,secrets,sa,configmaps,deployments,ingresses,clusterroles,clusterrolebindings,customresourcedefinitions --selector=app=kiali -n istio-system
```