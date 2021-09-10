



__Install kapp-controller Components__

Use kapp to install kapp-controller (reconciliation may take a moment, which you could use to read about kubernetes controller reconciliation loops):

```execute
kapp deploy -a kc -f https://github.com/vmware-tanzu/carvel-kapp-controller/releases/download/v0.21.0/release.yml -y
```

Observe the installed components 

```execute 
kubectl get all -n kapp-controller
```
The kapp deployment is managing a replicaset which owns a service and a pod. The pod is running kapp-controller, which is a kubernetes controller running its own reconciliation loop.

kapp-controller introduces new Custom Resource (CR) types we’ll use throughout this tutorial, including PackageRepositories and PackageInstalls.
You can checkout the API resources using the below command

```execute
kubectl api-resources --api-group packaging.carvel.dev
```

You can see other kapp-controller CRs in other groups:

```execute
kubectl api-resources --api-group data.packaging.carvel.dev
```
```execute
kubectl api-resources --api-group kappctrl.k14s.io
```

