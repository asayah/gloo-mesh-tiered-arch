# Gloo Mesh Tiered Architecture
Example of a Gloo Mesh deployment, where multiple Istio components are operated by different teams


# Architecture 

 ![Arch](https://raw.githubusercontent.com/asayah/gloo-mesh-tiered-arch/main/arch.png)


# End Goal

Using A simple CR created by the application team to expose their service securely in a Tiered architecture. 


# Installation 

## Prerequisites

To demonstrate the tiered architecture we will need 3 different clusters, although one or two can be used, I recommend using 3 cluster just to see how things are structured for this demo. 

Make sure you have 3 contexts, named : **mgmt**, **cluster1**, **cluster2**,  **mgmt** will be used as a management cluster where we will install Gloo Mesh control plane, **cluster1** will be our Gateway cluster, this is the one exposed to the internet, **cluster2** is our workload cluster where our application is installed. 

These values are hardcoded, if you want use different names, just update the scripts accordingly. 

## Istio Installation

run the following script: 

```bash
./install-istio.sh 
```

This will install Istio in the **cluster1** and **cluster2**, **cluster1** will have a north south gateway + and east west gateway, **cluster2** will have only an east west gateway.  
 
## Gloo Mesh Installation

Run the following script to install Gloo Mesh in the management cluster **mgmt**

```bash
./install-gloo-mesh.sh
```

# The Setup 

Now that we have the different components installed lets start setting up our environment. 

The Admin will create workspaces to restrict the other team so they can operate only their components, The gateway team will be able to just configure the gateways, and the app team will be able to just configure the app. 

## The Admin Team 

As an Admin I want first to unified the trust across my clusters (so the mtls will work seamlessly from a cluster to another), apply the following config in the me management cluster: 

```bash 
kubectl apply -f ./admin-team/root-trust.yaml --context mgmt -n gloo-mesh 
```
Now as an Admin, I will create workspaces for the teams. 

```
kubectl apply -f ./admin-team/workspaces --context mgmt -n gloo-mesh
```
you will see two workspaces created; gateways (gateway team) and httpbin (app team)

## The Gateway Team

Now it's time for the gateway team to manage the gateways, for that they will create a VirtualGateway in the gateways workspace. 

```bash
kubectl apply -f ./gateway-team --context cluster1 -n istio-gateways
```
Note that that WorkspaceSettings creating is exporting the configuration to any workspace label with allow_ingress=true, allowing the application team to import the gateway (discovery) 

## The Application Team

Now its time for the application team to expose their service (httpbin), for this they will create a WorkspaceSettings to import the gateways: 

```bash
kubectl create ns httpbin --context cluster2
kubectl apply -f ./application-team/workspace-settings.yaml --context cluster2 -n httpbin
```

Now they have just to deploy the application service itself (they can do that earlier too):

```
kubectl --context cluster2 label namespace httpbin istio.io/rev=1-13
kubectl apply -f ./application-team/httpbin.yaml --context cluster2 -n httpbin
```

And finally they can deploy this application though the gateway by creating this simple CR: 

```
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: httpbin
  namespace: httpbin
  labels:
    expose: "true"
spec:
  hosts:
    - '*'
  virtualGateways:
    - name: north-south-gw
      namespace: istio-gateways
      cluster: cluster1
  workloadSelectors: []
  http:
    - name: httpbin
      matchers:
      - uri:
          prefix: /
      forwardTo:
        destinations:
          - ref:
              name: httpbin
              namespace: httpbin
              cluster: cluster2
            port:
              number: 8000

```

Apply the CR described above using this: 

```bash
kubectl apply -f ./application-team/route-table.yaml --context cluster2 -n httpbin
```


Now you are all set, using the North-South Ip of the gateway living in cluster1, you can access your httpbin application living in cluster2. 


#TODO 

- Example using an external service (outside of Kubernetes), ref: https://github.com/solo-io/workshops/blob/master/gloo-mesh-2-0/README.md#Lab-15
- Example of OIDC + RBAC, ref: https://github.com/solo-io/workshops/blob/master/gloo-mesh-2-0/README.md#lab-16---deploy-keycloak-