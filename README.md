The use case is to deploy your local source code into open shift using a custom base image.
### So how can we achieve it?
This will require multiple steps as follows:

#### 1 Create an image stream with a tag that holds the base image:

Import base and runtime Images from upstream and schedule them to be updated automatically. We will cache layers of these big images in the local registry to make image pull faster for development as well as runtime applications.

> **Good Practice**: Create these image streams in a shared project/namespace and then give access to them as required.

CUDA Base:
```bash
oc new-project image-source

oc import-image cuda-devel --confirm --schedule=true --reference-policy local --from docker.io/nvidia/cuda:11.8.0-cudnn8-devel-ubi8
```

Generated IS:
```yaml
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  creationTimestamp: null
  name: cuda-devel
spec:
  lookupPolicy:
    local: false
  tags:
  - annotations: null
    from:
      kind: DockerImage
      name: docker.io/nvidia/cuda:11.8.0-cudnn8-devel-ubi8
    generation: null
    importPolicy: {}
    name: ""
    referencePolicy:
      type: Local
status:
  dockerImageRepository: ""
```

CUDA Runtime:
```bash
oc import-image cuda-runtime --confirm --scheduled=true --reference-policy local --from docker.io/nvidia/cuda:11.8.0-cudnn8-runtime-ubi8 -o yaml
```

Generated IS:
```yaml
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  creationTimestamp: null
  name: cuda-runtime
spec:
  lookupPolicy:
    local: false
  tags:
  - annotations: null
    from:
      kind: DockerImage
      name: docker.io/nvidia/cuda:11.8.0-cudnn8-runtime-ubi8
    generation: null
    importPolicy: {}
    name: ""
    referencePolicy:
      type: Local
status:
  dockerImageRepository: ""
```

#### 2 Prepare the custom image you want to use for your work.

Most of the time users will need to apply some customization on top of these base images. They should be able to use image streams to create a build config and apply their customizations on top of it.

> Let's take an example here, where we will modify the Developer CUDA image and open up sudo access. Which is dangerous and should be avoided but some users really need it for their work to happen before they finalize their final image for production.

#1 Setup Project/Namespace:
```bash
oc new-project user1-app1

# Give access to the Service Account from this namespace to image streams in another # project/namespace
# The system:image-puller role allows a service account to pull the image layers that the image stream cached in the internal registry.
# The system:serviceaccounts:user1-app1 group includes all service accounts from the user1-app1 project.

oc policy add-role-to-group system:image-puller system:serviceaccounts:user1-app1
```

#2 Create a build config with Docker as a strategy to provide a customized base image for your source code testing.

```bash
# Note the use of image stream with namespace before the name of the image stream

oc new-build --name custom-image-build --binary --strategy docker --image-stream image-source/cuda-devel:latest

#This will create an image stream and build config in user1-app1 namespace with a binary source of Dockerfile. 
```

Generated resources:
```bash
--> Found image 448cf64 (2 weeks old) in image stream "image-source/cuda-devel" under tag "latest" for "image-source/cuda-devel:latest"

    Red Hat Universal Base Image 8
    ------------------------------
    The Universal Base Image is designed and engineered to be the base layer for all of your containerized applications, middleware and utilities. This base image is freely redistributable, but Red Hat only supports Red Hat technologies through subscriptions for Red Hat products. This image is maintained by Red Hat and updated regularly.

    Tags: base rhel8

    * A source build using binary input will be created
      * The resulting image will be pushed to image stream tag "custom-image-build:latest"
      * A binary build was created, use 'oc start-build --from-dir' to trigger a new build

--> Creating resources with label build=custom-image-build ...
    imagestream.image.openshift.io "custom-image-build" created
    buildconfig.build.openshift.io "custom-image-build" created
--> Success
```

Now let's start the build with our customizations in the Docker file.

```bash
cd to_your_folder_where_dockerfile_is_present
oc start-build custom-image-build -F --from-dir=.
```

Above will start a build with all the customization you wanted in your image and push it to the local registry in OpenShift. You can later push this image to any external registry manually or modify BuildConfig to push it to the remote registry automatically every time the build runs.

Example:
```bash
....
....
Successfully pushed image-registry.openshift-image-registry.svc:5000/user1-app1/custom-image-build@sha256:072f4e8199e060a567687931c87d9554ebc30a08c130baa1752d4ae6af6133ff
Push successful
```

#### 3 Prepare source code for Source to Image(S2I) Deployment

To use custom images for your source-to-image deployment, you will need to create 2 files `assemble` and `run` in your source code directory under the path `.s2i/bin/` from your root directory.

> Here we are assuming your base or custom image is capable of detecting these files which is true for most of the UBI base images. If not then you will need to modify your custom image as mentioned here: https://github.com/openshift/source-to-image/blob/master/docs/builder_image.md#s2i-scripts

Create an `assemble` as the below that can install the required packages for your applications.
```bash

```
Create a `run` file to run the Python source code.
```bash

```

#### 4 Create a Build Config with a binary source strategy

Now we need to create another build config that will use our custom image produced earlier and take source code from the local machine or from GitHub and prepare an image to run.

```bash
# Note the use of image stream with namespace before the name of the image stream

oc new-build --name app1-dev-build --binary --strategy source --image-stream custom-image-build:latest

#This will create an image stream and build config in user1-app1 namespace with a binary source which we will need to provide every time we kick off a build and push the resulting image to the local image stream tag. 
```

> Using this strategy, developers won't need to create their images from scratch every time they need to deploy and test their code.

Generated Imagestream and buildconfig:
```yaml
....
....
    * A source build using binary input will be created
      * The resulting image will be pushed to image stream tag "app1-dev-build:latest"
      * A binary build was created, use 'oc start-build --from-dir' to trigger a new build

--> Creating resources with label build=app1-dev-build ...
    imagestream.image.openshift.io "app1-dev-build" created
    buildconfig.build.openshift.io "app1-dev-build" created
--> Success
```

#### 5 Create a build using local source code on the custom image

Let's take the build config created earlier and start a build using the code available locally on your workstation. Remember to cd into your source code directory.

```bash
oc start-build app1-dev-build -F --from-dir=.
```

This will create an image with your source code on the custom image that you created earlier.

```bash
Copying config sha256:0a885d530abcdb551e537417799fee85eabffdf9be2ed11f44396ff7e2a3bd2c
Writing manifest to image destination
Storing signatures
Successfully pushed image-registry.openshift-image-registry.svc:5000/user1-app1/app1-dev-build@sha256:5b92ed693d1d7065e021cbe13431c5691f61fbc2ffed210ca78ce01c64ec70ed
Push successful
```

#### 6 Deploy your source code using the image created earlier

oc new-app app1-dev-build.
