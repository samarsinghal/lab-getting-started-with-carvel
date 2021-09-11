Now that we are done with the install of the kapp-controller on the cluster. Let us work on authoring a package that we will use the controller to manager. 

__Creating a package__
- Configuration
For this demo, we will be using ytt templates that describe a simple Kubernetes Deployment and Service. These templates will install a simple greeter app with a templated hello message. The templates consist of two files:

config.yml:
```yml
#@ load("@ytt:data", "data")

#@ def labels():
simple-app: ""
#@ end

---
apiVersion: v1
kind: Service
metadata:
  name: simple-app
spec:
  ports:
  - port: #@ data.values.svc_port
    targetPort: #@ data.values.app_port
  selector: #@ labels()
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-app
spec:
  selector:
    matchLabels: #@ labels()
  template:
    metadata:
      labels: #@ labels()
    spec:
      containers:
      - name: simple-app
        image: docker.io/dkalinin/k8s-simple-app@sha256:4c8b96d4fffdfae29258d94a22ae4ad1fe36139d47288b8960d9958d1e63a9d0
        env:
        - name: HELLO_MSG
          value: #@ data.values.hello_msg
```

and values.yml:

```yml
#@data/values
---
svc_port: 80
app_port: 80
hello_msg: stranger
```

You will find these files in the `config-step-7-kapp-controller/config/` folder

__Package Contents Bundle__
The first step in creating our package is to create an imgpkg bundle that contains the package contents: the above configuration (config.yml and values.yml) and a reference to the greeter app image (docker.io/dkalinin/k8s-simple-app@sha256:...).

To start, lets go into the config folder `config-step-7-kapp-controller/config`

Let’s use kbld to record which container images are used:

```execute
mkdir -p config-step-7-kapp-controller/config/.imgpkg
```

```execute 
kbld -f config-step-7-kapp-controller/config --imgpkg-lock-output config-step-7-kapp-controller/config/.imgpkg/images.yml
```
The above steps uses the imgpkg tool that you learned about in the earlier sections. 

Once these files have been added, our package contents bundle is ready to be pushed as shown below (NOTE: replace registry.corp.com/packages/ if working through example):

```execute ---Need to change the registry info here. 
imgpkg push -b registry.corp.com/packages/simple-app:1.0.0 -f config-step-7-kapp-controller/
dir: .
file: .imgpkg/images.yml
file: config/config.yml
file: config/values.yml
Pushed 'registry.corp.com/packages/simple-app@sha256:e6255cc...'
Succeeded
```

Creating the Custom resources 

Now that the image is packaged we need to create the Custom Resources that refer to it. 

You will find the yaml files for these CRs in the `config-step-7-kapp-controller` folder. 

A package definition comprises of 2 Custom resources. 
- The `Package Metadata` which will contain high level information and descriptions about our package ( key description and info about the package)
```yml
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
```

- The `Package`containing versioned instructions and metadata used to install packaged software that fits the description provided in the PackageMetadata CR). You will see that the package definition has instructions for `ytt` , `kbld` and the `kapp` command that you worked on in the earlier sections. 

```yml
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
          image: registry.corp.com/packages/simple-app:1.0.0
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
```

__Testing your package__

Now that we have our package defined, we can test it on the cluster. We will momentarily act as a package consumer. First, we need to make our package available on the cluster, so let’s apply the Package and PackageMetadata CRs we just created directly to the cluster:

```execute 
kapp deploy -a package -f config-step-7-kapp-controller/1.0.0.yml -f config-step-7-kapp-controller/metadata.yml -y
```

The carvel docs has more information on how to verify the functioning of the package as well as debugging the installed package when you hit roadblocks. 

