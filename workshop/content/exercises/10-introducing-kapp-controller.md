# Introducing kapp-controller

## About

[kapp-controller](https://carvel.dev/kapp-controller/) provides a Kubernetes native [continuous delivery](https://carvel.dev/kapp-controller/docs/latest/app-spec/) and [package management])(https://carvel.dev/kapp-controller/docs/latest/packaging/) experience through custom resource definitions. These new resources for continuous delivery and package management help users author software packages and consume packages to ease the process of sharing, deploying, and managing software on Kubernetes.

Given that software configurations for Kubernetes software can be specified in various forms:

* plain [YAML](https://yaml.org/spec/1.2/spec.html) configurations
* [Helm charts](https://helm.sh/docs/topics/charts/)
* [ytt](http://carvel.dev/ytt/) templates
* [jsonnet](https://jsonnet.org/learning/tutorial.html) templates

and found in various locations:

* Git repository
* Archive over HTTP
* Helm repository

and written/provided by:

* in-house development teams
* vendors offering COTS products

kapp-controller allows users to encapsulate, customize, install, and update such software in a _consistent_ and _manageable_ manner.

Another motivation for kapp-controller was to make a small and single purpose system (as opposed to a general CD system); hence, it’s lightweight, easy-to-understand and easy-to-debug. It builds on small composable tools to achieve its goal and therefore is easy to think about.

Finally, for the fans of GitOps, kapp-controller turns [kapp](https://carvel.dev/kapp/) into your continuous cluster reconciler.

## Installation

Consult this [guide](https://carvel.dev/kapp-controller/docs/latest/install/) for how to install `kapp-controller` in your own cluster.

We've already installed kapp-controller in the cluster as part of the `startup.sh` script we invoked earlier.

Let's inspect it

```execute
kapp inspect -a kc -t --yes
```

## Configuration for continuous deployments

kapp-controller introduces a Kubernetes custom resource definition called `App` and the specification is broken out into three parts

* [fetch](https://carvel.dev/kapp-controller/docs/latest/config/#specfetch)
  * Fetches a set of files from various sources
* [template](https://carvel.dev/kapp-controller/docs/latest/config/#spectemplate)
  * Transforms a set of files
* [deploy](https://carvel.dev/kapp-controller/docs/latest/config/#specdeploy)
  * Deploys resources

Let's look at an example

```
apiVersion: kappctrl.k14s.io/v1alpha1
kind: App
metadata:
  name: simple-app
spec:
  serviceAccountName: default
  fetch:
  - git:
      url: https://github.com/vmware-tanzu/carvel-simple-app-on-kubernetes
      ref: origin/develop
      subPath: config-step-2-template
  template:
  - ytt: {}
  deploy:
  - kapp: {}
```

> Here we are simply _fetching_ then _deploying_ an application (consisting of [Service](https://kubernetes.io/docs/concepts/services-networking/service/#defining-a-service) and [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment) resources) from a particular sub-directory within a Git repository.

With this useful construct we can configure an instance of kapp-controller deployed within a cluster to watch for (and fetch) updates to one or more Git repositories or any other supported sources.

## Packaging for authors and consumers

kapp-controller also adds new APIs to bring common package management workflows to a Kubernetes cluster. This is done using four new CRs:

* [PackageRepository](https://carvel.dev/kapp-controller/docs/latest/packaging/#packagerepository)
* [PackageMetadata](https://carvel.dev/kapp-controller/docs/latest/packaging/#packagemetadata)
* [Package](https://carvel.dev/kapp-controller/docs/latest/packaging/#package-1)
* [PackageInstall](https://carvel.dev/kapp-controller/docs/latest/packaging/#packageinstall)

_Package Authors_ can publish bundles in two formats:

* [Package Contents Bundle](https://carvel.dev/kapp-controller/docs/latest/packaging-artifact-formats/#package-contents-bundle)
* [Package Repository Bundle](https://carvel.dev/kapp-controller/docs/latest/packaging-artifact-formats/#package-repository-bundle)

And _Package Consumers_ may select packages for namespace or cluster-wide deployment (i.e, those packages treated as global namespace) and handle potential [collisions](https://carvel.dev/kapp-controller/docs/latest/package-consumer-concepts/#collisions) by overriding the namespace for deployment in PackageMetadata definitions.  Be sure to take a look at the [Annotations](https://carvel.dev/kapp-controller/docs/latest/package-consumer-concepts/#annotations) feature too.  Lastly, consumers may [choose what version](https://carvel.dev/kapp-controller/docs/latest/package-consumer-concepts/#constraints) of a package to install using a PackageInstall constraint definition.

## Playtime

We will be using [ytt](https://carvel.dev/ytt/) templates that describe a simple Kubernetes Deployment and Service. These templates will install a simple greeter app with a templated hello message. (You've seen this config before).

### Creating a Package: Structuring our contents

We’ll create an [imgpkg](https://carvel.dev/imgpkg/) bundle that contains the package contents: the configuration (config.yml and values.yml from the previous step) and a reference to the greeter app image

The [package bundle format](https://carvel.dev/kapp-controller/docs/latest/packaging/#package-bundle-format) describes the purpose of each directory used in this section of the tutorial as well as general recommendations.

Let’s create a directory with our configuration files:

```execute
mkdir -p package-contents/config/
cp config-step-3-build-local/config.yml package-contents/config/config.yml
cp config-step-3-build-local/values.yml package-contents/config/values.yml
```

Once we have the configuration figured out, let’s use [kbld](https://carvel.dev/kbld/) to record which container images are used:

```execute
mkdir -p package-contents/.imgpkg
kbld -f package-contents/config/ --imgpkg-lock-output package-contents/.imgpkg/images.yml
```

For more on using kbld to populate the .imgpkg directory with an ImagesLock, and why it is useful, see the [imgpkg docs on the subject](https://carvel.dev/imgpkg/docs/latest/resources/#imageslock-configuration).

Once these files have been added, our package contents bundle is ready to be pushed!

```execute
imgpkg push -b $registry/simple-app:1.0.0 -f package-contents/
```

### Creating the Custom Resources

To finish creating a package, we need to create two CRs. The first CR is the PackageMetadata CR, which will contain high level information and descriptions about our package.

When creating this CR, the api will validate that the PackageMetadata’s name is a fully qualified name: It must have at least three segments separated by . and cannot have a trailing ..

We’ll make a conformant `metadata.yml` file:

```execute
cat > metadata.yml << EOF
apiVersion: data.packaging.carvel.dev/v1alpha1
kind: PackageMetadata
metadata:
  # This will be the name of our package
  name: simple-app.corp.com
spec:
  displayName: "Simple App"
  longDescription: "Simple app consisting of a k8s deployment and service"
  shortDescription: "Simple app for demoing"
  categories:
  - demo
EOF
```

Now we need to create a Package CR. This CR contains versioned instructions and metadata used to install packaged software that fits the description provided in the PackageMetadata CR we just saved in `metadata.yml`.

```execute
cat > 1.0.0.yml << EOF
---
apiVersion: data.packaging.carvel.dev/v1alpha1
kind: Package
metadata:
  name: simple-app.corp.com.1.0.0
spec:
  refName: simple-app.corp.com
  version: 1.0.0
  releaseNotes: |
        Initial release of the simple app package
  valuesSchema:
    openAPIv3:
      title: simple-app.corp.com values schema
      examples:
      - svc_port: 80
        app_port: 80
        hello_msg: stranger
      properties:
        svc_port:
          type: integer
          description: Port number for the service.
          default: 80
          examples:
          - 80
        app_port:
          type: integer
          description: Target port for the application.
          default: 80
          examples:
          - 80
        hello_msg:
          type: string
          description: Name used in hello message from app when app is pinged.
          default: stranger
          examples:
          - stranger
  template:
    spec:
      fetch:
      - imgpkgBundle:
          image: $registry/simple-app:1.0.0
      template:
      - ytt:
          paths:
          - "config/"
      - kbld:
          paths:
          - "-"
          - ".imgpkg/images.yml"
      deploy:
      - kapp: {}
EOF
```

This Package contains some metadata fields specific to the version, such as releaseNotes and a valuesSchema. The valuesSchema shows what configurable properties exist for the version. This will help when users want to install this package and want to know what can be configured.

The other main component of this CR is the template section. This section informs kapp-controller of the actions required to install the packaged software, so take a look at the app-spec section to learn more about each of the template sections. For this example, we have chosen a basic setup that will fetch the imgpkg bundle we created in the previous section, run the templates stored inside through ytt, apply kbld transformations, and then deploy the resulting manifests with kapp.

There will also be validations run on the Package CR, so ensure that spec.refName and spec.version are not empty and that metadata.name is <spec.refName>.<spec.version>. These validations are done to encourage a naming scheme that keeps package version names unique.

### Creating a Package Repository

A [package repository bundle](https://carvel.dev/kapp-controller/docs/latest/packaging/#package-repository-bundle-format) is a collection of packages (more specifically a collection of Package and PackageMetadata CRs). Currently, our recommended way to make a package repository is via an [imgpkg bundle](https://carvel.dev/imgpkg/docs/latest/resources/#bundle).

The [PackageRepository bundle format](https://carvel.dev/kapp-controller/docs/latest/packaging/#package-repository-bundle-format) describes purpose of each directory and general recommendations.

Let's start by creating the needed directories:

```execute
mkdir -p my-pkg-repo/.imgpkg my-pkg-repo/packages/simple-app.corp.com
```

we can copy our CR YAMLs from the previous step in to the proper packages subdirectory:

```execute
cp 1.0.0.yml my-pkg-repo/packages/simple-app.corp.com
cp metadata.yml my-pkg-repo/packages/simple-app.corp.com
```

Next, let’s use kbld to record which package bundles are used:

```execute
kbld -f my-pkg-repo/packages/ --imgpkg-lock-output my-pkg-repo/.imgpkg/images.yml
```

With the bundle metadata files present, we can push our bundle to whatever OCI registry we plan to distribute it from, which for this tutorial will just be Harbor.

```execute
imgpkg push -b $registry/my-pkg-repo:1.0.0 -f my-pkg-repo
```

The package repository is pushed!

In the next steps we’ll act as the package consumer, showing an example of adding and using a PackageRepository with kapp-controller.

### Adding a PackageRepository

kapp-controller needs to know which packages are available to install. One way to let it know about available packages is by creating a package repository. To do this, we need a PackageRepository CR:

```execute
cat > repo.yml << EOF
---
apiVersion: packaging.carvel.dev/v1alpha1
kind: PackageRepository
metadata:
  name: simple-package-repository
spec:
  fetch:
    imgpkgBundle:
      image: $registry/my-pkg-repo:1.0.0
EOF
```

This PackageRepository CR will allow kapp-controller to install any of the packages found within the `$registry/my-pkg-repo:1.0.0` imgpkg bundle, which we stored in our Harbor registry previously.

We can use kapp to apply it to the cluster:

```execute
kapp deploy -a repo -f repo.yml -y
```

Check for the success of reconciliation to see the repository become available:

```execute
kubectl get packagerepository -w
```

Once the simple-package-repository has a __Reconcile succeeded__ description, we’re ready to continue! You can exit the watch by pressing `Ctrl-c`.

Once the deploy has finished, we are able to list the package metadatas to see, at a high level, which packages are now available within our namespace:

```execute
kubectl get packagemetadatas
```

If there are numerous available packages, each with many versions, this list can become a bit unwieldy, so we can also list the packages with a particular name using the –field-selector option on kubectl get.

```execute
kubectl get packages --field-selector spec.refName=simple-app.corp.com
```

From here, if we are interested, we can further inspect each version to discover information such as release notes, installation steps, licenses, etc. For example:

```execute
kubectl get package simple-app.corp.com.1.0.0 -o yaml
```

### Installing a Package

Once we have the packages available for installation (as seen via kubectl get packages), we need to let kapp-controller know which package we want to install. To do this, we will need to create a PackageInstall CR (and a secret to hold the values used by our package):

```execute
cat > pkginstall.yml << EOF
---
apiVersion: packaging.carvel.dev/v1alpha1
kind: PackageInstall
metadata:
  name: pkg-demo
spec:
  serviceAccountName: default-ns-sa
  packageRef:
    refName: simple-app.corp.com
    versionSelection:
      constraints: 1.0.0
  values:
  - secretRef:
      name: pkg-demo-values
---
apiVersion: v1
kind: Secret
metadata:
  name: pkg-demo-values
stringData:
  values.yml: |
    ---
    hello_msg: "to all my Spring One friends"
EOF
```

This CR references the Package we created in the previous sections using the package's `refName` and `version` fields. Do note, the `versionSelection` property has a constraints subproperty to give more control over which versions are chosen for installation. More information on PackageInstall versioning can be found [here](https://carvel.dev/kapp-controller/docs/latest/packaging/#versioning-packageinstalls).

This yaml snippet also contains a Kubernetes secret, which is referenced by the PackageInstall. This secret is used to provide customized values to the package installation’s templating steps. Consumers can discover more details on the configurable properties of a package by inspecting the Package CR’s valuesSchema.

Finally, to install the above package, we will also need to create `default-ns-sa` service account (refer to [Security model](https://carvel.dev/kapp-controller/docs/latest/security-model/) for explanation of how service accounts are used) that give kapp-controller privileges to create resources in the default namespace:

```execute
kapp deploy -a default-ns-rbac -f https://raw.githubusercontent.com/vmware-tanzu/carvel-kapp-controller/develop/examples/rbac/default-ns.yml -y
```

Apply the PackageInstall using kapp:

```execute
kapp deploy -a pkg-demo -f pkginstall.yml -y
```

After the deploy has finished, kapp-controller will have installed the package in the cluster. We can verify this by checking the pods to see that we have a workload pod running. The output should show a single running pod which is part of simple-app:

```execute
kubectl get pods
```

At this point you could set up port-forwarding and verify the customized hello message.  See if you can figure out how to do that yourself.
