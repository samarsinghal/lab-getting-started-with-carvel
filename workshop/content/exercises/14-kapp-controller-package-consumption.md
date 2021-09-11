__kapp-controller package consumption__

Now that we learnt how to create the package, we need to understand how the carvel tooling allows you to make it easy for software to be easily discovered and installed. 

The first Custom resource used for this is the `PackageRepository`. 

The `config-step-7-kapp-controller/repository.yml` in the folder has a sample definiton pointing to an image bundle.
```yml
apiVersion: packaging.carvel.dev/v1alpha1
kind: PackageRepository
metadata:
  name: simple-package-repository
spec:
  fetch:
    imgpkgBundle:
      image: k8slt/corp-com-pkg-repo:1.0.0
```

Let us use this file to create a repository on our cluster. 

```execute
kubectl apply -f config-step-7-kapp-controller/repository.yml
```

Once the Repository is installed,  we can observe the available packages from the repository for installation. 

```execute
kubectl get packages
```

```
NAME                             PACKAGEMETADATA NAME   VERSION      AGE
simple-app.corp.com.1.0.0        simple-app.corp.com    1.0.0        1m50s
simple-app.corp.com.2.0.0        simple-app.corp.com    2.0.0        1m50s
simple-app.corp.com.3.0.0-rc.1   simple-app.corp.com    3.0.0-rc.1   1m50s
```

Having now discovered the available packages, kapp-controller's PackageInstall CR provides users a way to install Packages on a Kubernetes cluster.

Let us use the `config-step-7-kapp-controller/repository.yml` to deploy the software in the package on the cluster. 

```execute 
kubectl apply -f config-step-7-kapp-controller/packageinstall.yml
```

Observe the installed package

```execute 
kubectl get packageinstall pkg-demo
```

```
NAME       PACKAGE NAME          PACKAGE VERSION   DESCRIPTION           AGE
pkg-demo   simple-app.corp.com   1.0.0             Reconcile succeeded   1m50s
```

Once the reconciliation is complete,  you will be able to observe the app ( kapp's app ) that is installed 

```execute 
$ kubectl get app pkg-demo
```

```
NAME         DESCRIPTION           SINCE-DEPLOY   AGE
pkg-demo     Reconcile succeeded   9s             1m45s
```

