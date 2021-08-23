# Packaging bundles for distribution

We're going to explore a couple of scenarios where [imgpkg](https://carvel.dev/imgpkg/) may assist in the packaging and relocation of [OCI](https://opencontainers.org/) images and [bundles](https://github.com/opencontainers/runtime-spec#application-bundle-builders).  Then we'll take a crack at deploying a bundle using `kbld` and `kapp`.

## Scenario 1: Basic workflow

You want to create an immutable artifact containing Kubernetes configuration and images used in that configuration. Later, you want to grab that artifact and deploy it to Kubernetes.

### Step 1: Creating the bundle

The config-step-6a-bundle-basic-workflow directory has a config.yml file, which contains a very simple Kubernetes application. Your application may have as many configuration files as necessary in various formats such as plain YAML, ytt templates, Helm templates, etc.

To view it type

```
cat config-step-6a-bundle-basic-workflow/config.yml
```

It includes an image reference to `quay.io/eduk8s-labs/sample-app-go`. This reference does not point to an exact image (via digest) meaning that it may change over time. To ensure we get precisely the bits we expect, we will lock it down to an exact image next.

Let's take a look inside the `.imgpkg` directory

```
tree config-step-6a-bundle-basic-workflow
```

It contains:

* an optional [bundle.yml](https://carvel.dev/imgpkg/docs/latest/resources/#bundle-metadata)
  * a file which records informational metadata
* a required [images.yml](https://carvel.dev/imgpkg/docs/latest/resources/#imageslock)
  * a file which records image references used by the configuration

Note that `.imgpkg/images.yml` contains a list of images, each with fully resolved digest reference and a little bit of additional metadata (e.g. annotations section). See [ImagesLock configuration](https://carvel.dev/imgpkg/docs/latest/resources/#imageslock-configuration) for details.

Let's take a look at the contents of `.imgpkg/images.yml`

```
cat config-step-6a-bundle-basic-workflow/.imgpkg/images.yml
```

You should see something like:

```
---
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: ImagesLock
images:
- image: quay.io/eduk8s-labs/sample-app-go@sha256:5021a23e0c4a4633bfd6c95b13898cffb88a0e67f109d87ec01b4f896f4b4296
  annotations:
    kbld.carvel.dev/id: quay.io/eduk8s-labs/sample-app-go
```

This allows us to record the exact image that will be used by our Kubernetes configuration. We expect that `.imgpkg/images.yml` would be created either manually, or in an automated way. Our recommendation is to use kbld to generate `.imgpkg/images.yml`:

```
kbld -f config-step-6a-bundle-basic-workflow/config.yml --imgpkg-lock-output config-step-6a-bundle-basic-workflow/.imgpkg/images.yml
```

### Step 2: Pushing the bundle to a registry

We've already authenticated with a registry where we will push our bundle.

You can push the bundle with our specified contents to Harbor (or any other OCI compliant container image registry) using the following command:

```
imgpkg push -b core.harbor.domain/library/simple-app-bundle:v1.0.0 -f config-step-6a-bundle-basic-workflow
```

You should see output like:

```
dir: .
dir: .imgpkg
file: .imgpkg/bundle.yml
file: .imgpkg/images.yml
file: config.yml
Pushed 'core.harbor.domain/library/simple-app-bundle@sha256:91bc50768d10db3850be5a6af67d43e3ca92d92c340b19d7d18574e447fd74ae'
Succeeded
```

### Step 3: Pulling the bundle to registry

Now that we have pushed a bundle to a registry, other users can pull it.

A user would authenticate with the registry then pull our bundle.

We're already authenticated so to download the bundle run the following command:

```
imgpkg pull -b core.harbor.domain/library/simple-app-bundle:v1.0.0 -o /tmp/simple-app-bundle
```

You should see output like:

```
Pulling bundle 'core.harbor.domain/library/simple-app-bundle@sha256:91bc50768d10db3850be5a6af67d43e3ca92d92c340b19d7d18574e447fd74ae'
  Extracting layer 'sha256:0ed990f64175aa92f7f2fb5349dad79751b4fb1d451f6ec6511c6403822edfae' (1/1)

Locating image lock file images...
One or more images not found in bundle repo; skipping lock file update

Succeeded
```

Flags used in the command:

* `-b (--bundle)` refers to a location for a bundle within an OCI registry
* `-o (--output)` indicates the destination directory on your local machine where the bundle contents will be placed

View the bundle contents that were extracted into the `/tmp/simple-app-bundle` directory with:

```
tree /tmp/simple-app-bundle -a
```

Note: The message `One or more images not found in bundle repo; skipping lock file update` is expected, and indicates that `/tmp/simple-app-bundle/.imgpkg/images.yml` (ImagesLock configuration) was not modified.

If `imgpkg` had been able to find all images that were referenced in the `ImagesLock` configuration in the registry where bundle is located, then it would update `.imgpkg/images.yml` file to point to the `registry-local` locations.

### Step 4: Use pulled bundle contents

Now that we have have pulled bundle contents to a local directory, we can deploy Kubernetes configuration.

Before we apply Kubernetes configuration, let’s use kbld to ensure that Kubernetes configuration uses exact image reference from .imgpkg/images.yml. (You can of course use other tools to take advantage of data stored in .imgpkg/images.yml).

```
kbld -f /tmp/simple-app-bundle/config.yml -f /tmp/simple-app-bundle/.imgpkg/images.yml | kapp deploy -a simple-hello-app -f- --yes
```

You should see output like:

```
resolve | final: quay.io/eduk8s-labs/sample-app-go -> quay.io/eduk8s-labs/sample-app-go@sha256:5021a23e0c4a4633bfd6c95b13898cffb88a0e67f109d87ec01b4f896f4b4296
Target cluster 'https://127.0.0.1:37535' (nodes: carvel-v72626-control-plane)

Changes

Namespace  Name              Kind        Conds.  Age  Op      Op st.  Wait to    Rs  Ri
default    simple-hello-app  Deployment  -       -    create  -       reconcile  -   -
^          simple-hello-app  Service     -       -    create  -       reconcile  -   -

Op:      2 create, 0 delete, 0 update, 0 noop
Wait to: 2 reconcile, 0 delete, 0 noop

1:51:08AM: ---- applying 2 changes [0/2 done] ----
1:51:08AM: create deployment/simple-hello-app (apps/v1) namespace: default
1:51:08AM: create service/simple-hello-app (v1) namespace: default
1:51:08AM: ---- waiting on 2 changes [0/2 done] ----
1:51:08AM: ok: reconcile service/simple-hello-app (v1) namespace: default
1:51:08AM: ongoing: reconcile deployment/simple-hello-app (apps/v1) namespace: default
1:51:08AM:  ^ Waiting for generation 2 to be observed
1:51:08AM:  L ok: waiting on replicaset/simple-hello-app-69c4f9448 (apps/v1) namespace: default
1:51:08AM: ---- waiting on 1 changes [1/2 done] ----
1:51:08AM: ongoing: reconcile deployment/simple-hello-app (apps/v1) namespace: default
1:51:08AM:  ^ Waiting for generation 2 to be observed
1:51:08AM:  L ok: waiting on replicaset/simple-hello-app-69c4f9448 (apps/v1) namespace: default
1:51:08AM:  L ongoing: waiting on pod/simple-hello-app-69c4f9448-6cv8s (v1) namespace: default
1:51:08AM:     ^ Pending: ContainerCreating
1:51:09AM: ongoing: reconcile deployment/simple-hello-app (apps/v1) namespace: default
1:51:09AM:  ^ Waiting for 1 unavailable replicas
1:51:09AM:  L ok: waiting on replicaset/simple-hello-app-69c4f9448 (apps/v1) namespace: default
1:51:09AM:  L ongoing: waiting on pod/simple-hello-app-69c4f9448-6cv8s (v1) namespace: default
1:51:09AM:     ^ Pending: ContainerCreating
1:51:10AM: ok: reconcile deployment/simple-hello-app (apps/v1) namespace: default
1:51:10AM: ---- applying complete [2/2 done] ----
1:51:10AM: ---- waiting complete [2/2 done] ----

Succeeded
```

Above `kbld` found `quay.io/eduk8s-labs/sample-app-go` in Kubernetes configuration and replaced it with `quay.io/eduk8s-labs/sample-app-go@sha256:5021a23e0c4a4633bfd6c95b13898cffb88a0e67f109d87ec01b4f896f4b4296` before forwarding configuration to `kubectl`.

### Next steps

In this workflow we saw how to publish and download a bundle to distribute a Kubernetes application.

Next, we'll follow an Air-gapped workflow to see how we can use the `imgpkg copy` command to copy a bundle between registries.


## Scenario: Air-gapped workflow

You want to ensure Kubernetes application does not rely on images from external registries when deployed.

This scenario also applies when trying to ensure that all images are consolidated into a single registry, even if that registry is not air-gapped.

If any of your bundles contain non-distributable layers you will need to include the `--include-non-distributable-layers` flag to each copy command in the examples provided.


### Step 1: Copying bundles

You have two options when transferring an image or bundle from one registry to another

* From a common location connected to both registries.
  * This option is more efficient because only changed image layers will be transferred between registries.
* With intermediate tarball.
  * This option works best when registries have no common network access.

#### Option 1: From a location connected to both registries

You must be in a location that can access both registries.

This may be a server that has access to both internal and external networks. If there is no such location, you will have to use “Option 2” below.

You'd authenticate with both source and destination registries.

Run following command to copy an image from one registry to another:

```
imgpkg copy -i quay.io/opstree/redis-operator:0.8.0 --to-repo core.harbor.domain/library/redis-operator
```

You should see some output like:

```
copy | exporting 1 images...
copy | will export quay.io/opstree/redis-operator@sha256:c468940ff5bf700bd459e048c92ce9ecd751010162f591f0b14e80e3b39659a7
copy | exported 1 images
copy | importing 1 images...

 19.03 MiB / 19.04 MiB [=====================================================================================================================================================================================]  99.92% 7.65 MiB/s 2s

copy | done uploading images
Succeeded
```

A public hosted container image `quay.io/opstree/redis-operator:0.8.0` was copied to our private Harbor container image registry.

Flags used in the command:

* `-i (--image)` indicates the image location in the source registry
* `--to-repo` indicates the registry where the image should be copied to


#### Option 2: With intermediate tarball

Get to a location that can access the source registry.

Authenticate with the source registry (we've already done so).

Save the bundle to a tarball.

```
imgpkg copy -b core.harbor.domain/library/simple-app-bundle:v1.0.0 --to-tar /tmp/my-image.tar
```

You should see some output like:

```
copy | exporting 2 images...
copy | will export core.harbor.domain/library/simple-app-bundle@sha256:91bc50768d10db3850be5a6af67d43e3ca92d92c340b19d7d18574e447fd74ae
copy | will export quay.io/eduk8s-labs/sample-app-go@sha256:5021a23e0c4a4633bfd6c95b13898cffb88a0e67f109d87ec01b4f896f4b4296
copy | exported 2 images
copy | writing layers...
copy | done: file 'manifest.json' (7.38µs)
copy | done: file 'sha256-0ed990f64175aa92f7f2fb5349dad79751b4fb1d451f6ec6511c6403822edfae.tar.gz' (48.191µs)
copy | done: file 'sha256-58695640bb1f3fe15df3bc7fcd958c825e675ec773a5edaab12002c6b21bc23a.tar.gz' (523.962207ms)
Succeeded
```

Flags used in the command:

* `-b (--bundle)` indicates the bundle location in the source registry
* `--to-tar` indicates the local location to write a tar file containing the bundle assets

Transfer the local tarball `/tmp/my-image.tar` to a location with access to a destination registry.

Authenticate with the destination registry.

Import the bundle from your tarball to the destination registry:

```
imgpkg copy --tar /tmp/my-image.tar --to-repo core.harbor.domain/library/simple-app-bundle
```
> There's no real value in executing the above since the bundle already exists in the destination repository.  But if you had credentials and access to another private container image registry you could replace the `--to-repo` 

And so the bundle, and all images referenced in the bundle, would be copied to the destination registry.

Flags used in the command:

* `--tar` indicates the path to a tar file containing the assets to be copied to a registry
* `--to-repo` indicates destination bundle location in the registry

Regardless which location the bundle is downloaded from, source registry or destination registry, use of the pulled bundle contents remains the same.
You could continue with `Step 4: Use pulled bundle contents` in `Scenario 1: Basic workflow`.

