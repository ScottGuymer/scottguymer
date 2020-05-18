+++
categories = []
date = 2020-05-18T08:45:50Z
keywords = ["build", "containers", "security", "docker"]
tags = ["build", "containers", "security", "docker"]
title = "BlackDuck Docker image scanning from within a docker container"

+++

Docker is a great tool for build pipelines, without a doubt it allows you to create isolated and reproducible builds. Not just of docker images themselves, but also for artefacts that you might extract and use outside of the container.

When you are building Docker images security should be one of the concerns you can take care of in your CI pipeline. This varies from simple linting of your dockerfiles using something like [hadolint](https://github.com/hadolint/hadolint) to a more complex scanner that can scan the internals of your images and give you some clues as to where you might need to address security concerns.

Within my current organisation, we use a scanner called BlackDuck which among other things is able to scan both the source code of your application and any docker images.

In this post, we will look at using BlackDuck to scan your completed docker images.

I ran into quite a few challenges in getting the BlackDuck scan to work as part of the CI pipeline. To add extra difficulty I wanted to run the scanner from within its own docker container. This is simply because the only dependency installed on the build server should be docker itself. You can achieve most tasks with this philosophy but when it comes to running other containers it can get quite complex.

Put simply we wanted to start a BlackDuck scanner container and pass it an image reference and have it run its scanner over the indicated image, save the results locally so that they can be uploaded to the BlackDuck server later.

In the end, I ended up with this complex looking `docker run` command

```bash
docker run \
  -d \
  --network host \
  -v ~/.docker/config.json:/root/.docker/config.json \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /tmp/blackduck/inspectorshared/:/tmp/blackduck/inspectorshared/ \
  my.private.registry/blackduckscanner:${BLACKDUCK_DETECT_VERSION} \
    --detect.docker.passthrough.shared.dir.path.local=/tmp/blackduck/inspectorshared/ \
    --detect.docker.passthrough.shared.dir.path.imageinspector=/tmp/blackduck/inspectorshared/ \
    --detect.tools=DOCKER \
    --detect.project.name=${SCAN_PROJECT_NAME} \
    --detect.project.version.name=${SCAN_PROJECT_VERSION}_${SCAN_PROJECT_GROUP}_dockerImage \
    --detect.code.location.name=${SCAN_PROJECT_NAME}_${SCAN_PROJECT_GROUP}_${SCAN_PROJECT_VERSION}_${SCAN_COMPONENT_NAME}_dockerImage \
    --detect.bom.aggregate.name=${SCAN_COMPONENT_NAME} \
    --blackduck.offline.mode=true \
    --detect.docker.image=${IMAGE_PATH}
```

There is a lot to unpack in the above command.

- `-d` - run the image in the background rather than waiting
  I did this so that we can simply return the UUID for the container and wait for it to complete. We can then use `docker cp` to copy the scan results out of the container. This gets around permission issues created when mounting a volume to the running container. These are usually due to the container user being root and the host not being able to modify/delete files created inside the container.
- `--network host` set the networking mode to host so that all ports are accessible on `localhost` in the container
  We do this because the BlackDuck scanner starts up some other services that are used for scanning and by default it accesses them on localhost ports. This can be configured but this seems simpler.
- `~/.docker/config.json:/root/.docker/config.json`
  We then mount the host `config.json` to provide credentials for private registries. We assume that the host has already logged into the private registry that is used for the images we want to scan.
- `-v /var/run/docker.sock:/var/run/docker.sock`
  Mount the docker.sock so that the container can start containers on the docker host it is running on.
- `-v /tmp/blackduck/inspectorshared/:/tmp/blackduck/inspectorshared/`
  The Blackduck scanner uses mounted volumes to communicate with the other services that it starts. It's important that this volume is named the exact same inside the container and on the host.
- `my.private.registry/blackduckscanner:${BLACKDUCK_DETECT_VERSION}`
  The BlackDuck scanner image that we want to run that will complete the scans
- `-detect.docker.passthrough.shared.dir.path.local=/tmp/blackduck/inspectorshared/`
- `-detect.docker.passthrough.shared.dir.path.imageinspector=/tmp/blackduck/inspectorshared/`  
  These two properties work in conjunction with the volume mount above so that the volume is shared between the BlackDuck scanner container and the scanning services that the scanner starts as part of its operation. It's important that they match the volume defined above.
- `-detect.tools=DOCKER`  
  Tells the BlackDuck scanner to use the DOCKER tool, this is the tool used for scanning Docker images.
- `-blackduck.offline.mode=true`  
  For our purposes, we run the scanner in offline mode as we don't have direct access to the BlackDuck backend from our build environment.
- `-detect.docker.image=${IMAGE_PATH}`
  This property is where we specify the full path to the image that we want to have scanned. It must be a full path including the registry and tag e.g. `my.private.registry/myservice:v.10`, It should be pushed to the registry so that the scanner can pull to start the scan. It does not seem it will work with locally held images that already exist on the host.

The following are all BlackDuck properties that are used to identify the scan results created by the scanner.

- `-detect.project.name=${SCAN_PROJECT_NAME}`
- `-detect.project.version.name=${SCAN_PROJECT_VERSION}_${SCAN_PROJECT_GROUP}_dockerImage`
- `-detect.code.location.name=${SCAN_PROJECT_NAME}_${SCAN_PROJECT_GROUP}_${SCAN_PROJECT_VERSION}_${SCAN_COMPONENT_NAME}_dockerImage`
- `-detect.bom.aggregate.name=${SCAN_COMPONENT_NAME}`

## Getting the results our of the image

To bypass difficulties with mounting a volume to access the results directly we use a trick with `docker cp` to access the contents of the container after its completed its run

```bash
SCANNER_CONTAINER_UUID=$(docker run <snip>)
docker wait "${SCANNER_CONTAINER_UUID}"
docker cp "$SCANNER_CONTAINER_UUID:/root/blackduck" .
```
