This workshop environment provides you with a Kubernetes cluster to use when running the exercises covered by this workshop. To verify you can access the Kubernetes cluster, run `kubectl version` in the terminal by clicking on the action block below.

```execute
kubectl version
```

The workshop uses these action blocks for various purposes. Anytime you see such a block with an icon on the right hand side, you can click on it and it will perform the listed action for you.

To see what versions of the Carvel tools we will be using, you can run:

```execute
ytt --version
kbld --version
kapp --version
imgpkg --version
vendir --version
```


Login to the registry you would like to use

```execute
docker login
```

Now our local docker client is authenticated to the registry. Set registry environment variable 

```
export registry=
```

Also, we need to create an image pull secret to use with the deployment, so that Kubernetes can pull images from the private repo we are using:

```execute
kubectl create secret generic registry-credentials --from-file=.dockerconfigjson=$HOME/.docker/config.json --type=kubernetes.io/dockerconfigjson
```