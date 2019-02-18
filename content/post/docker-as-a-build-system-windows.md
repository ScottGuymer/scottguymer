+++
categories = []
date = "2018-06-14T20:09:50+01:00"
keywords = ["build", "containers", "windows", "docker"]
tags = ["docker", "windows", "containers", "build"]
title = "Docker as a build system - with Windows containers"

+++
> Whilst I am familiar with linux containers I havent had the chance to use windows containers much so this is part of a series where I explore the features and differences of windows container. These posts are written for someone who is fairly new to the docker experience.

Docker doesnt just have to be used just to build, package and deploy applications. Its ability to provide an isolated environment that you have control over is great for build systems in general.

Because we can start with a base image and describe the steps to get our application build including explicitly stating or installing any dependencies we have great control and knowledge of what are application truely needs to be built correctly. We can also leverage dockers layering and change detection to make our builds faster.

In this example im going to build a windows forms application that we would expect to use on a local desktop and not from running the image itself. In this case the applicaiton is trivial and is just a button that increments a counter each time you click it.

![image-20190215-100407.png](https://api.media.atlassian.com/file/bc3cf4f1-0baf-4444-a8b3-07f11f8d0b3b/artifact/image.jpg/binary?client=d869b65a-5c7c-4760-886d-1d80b45237f0&collection=contentId-747536502&max-age=3600&token=eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJkODY5YjY1YS01YzdjLTQ3NjAtODg2ZC0xZDgwYjQ1MjM3ZjAiLCJhY2Nlc3MiOnsidXJuOmZpbGVzdG9yZTpjb2xsZWN0aW9uOmNvbnRlbnRJZC03NDc1MzY1MDIiOlsicmVhZCJdfSwiZXhwIjoxNTUwNDgzMDM4LCJuYmYiOjE1NTA0ODAwOTh9.NRDUvwulOx4gSF-QXOm6yfbmDRFTA4cxrTZcFRKZuIY "clicker.exe")You can see the end result… Its not pretty but it works!

To start with we have a simple solution to build this application locally on our machine and everything works as expected. When we run the application it starts and we can click the button like crazy.

Now we have to think about getting this built inside a docker image. To start with we will need a base image with all the tools in that we need to be able to build a winforms app. This includes msbuild and various other tools. Instead of building this myself for simplicity I found one that will do the job on DockerHub. Youc an take a look at it here https://hub.docker.com/r/compulim/msbuild/ and the great thing about DockerHub is that most of the images are open sourece and you can usually find dockerfile that was used to create that image so you can inspect it for any funny stuff yourself. The repository for this one is on GitHub here https://github.com/compulim/docker-msbuild

The next task is to create a new dockerfile and add the steps needed to get the application build inside the container.

``` docker
FROM compulim/msbuild

WORKDIR c:\\code

ADD clicker .

RUN msbuild clicker.sln /p:Configuration=Debug /p:Platform="Any CPU"
```

This all looks pretty simple. We base our image on the msbuild one using the FROM directive, this means we are using that one as a starting point for our image. We then set a working directory using WORKDIR(this is optional but mean you are explicit about where things are going.) Next we simply ADD the code from our local disk into the docker image itself. And finally we direct the docker engine to RUNa command inside the docker image. This command will build the project we added and the output will be stored in the docker image.

We can get tht docker engine to build our docker image using the following command

``` bash
docker build -t winforms .
```

The first time we run this it will have to pull the image down from DockerHub which because windows images are very large can take some time. This only happens the first time. It will then start to run the instructions from the dockerfile above.

The result you get in the command line will look something like this

``` bash
Successfully built 327e142a1ac3
Successfully tagged winforms:latest
```

This results in a new image being created and stored on your local machine. You can see this by running docker images which will give you a list of the images available on your machine. You will notice that there are at least 2. The image we based everything on and your brand new image that we just built.

``` bash
REPOSITORY                             TAG                 IMAGE ID            CREATED             SIZE
winforms                               latest              327e142a1ac3        3 minutes ago       13.9GB
compulim/msbuild                       latest              5f5eb49a0f18        16 months ago       13.8GB
```

At the moment the exe that we just built is stuck inside your image which doesnt do us much good. Here is where the neat trick happens. Using docker we can create a new container from our image without starting it and then reach into that container and copy files back to our local system. The commands look like this.

``` bash
docker build -t winforms .

# create a new instance of our winforms image without starting it
docker create --name build winforms

# copy clicker.exe out of the container into our local filesystem
docker cp build:c:\code\bin\Debug\clicker.exe .

# clean up my removing the container we created
docker rm build
```

We now have the exe file locally and we can distribute it however we choose.

The benefit here is having a full description of the build environment that developers can reproduce locally and run the exact same build as the build system might do. We can include any dependencies or extra steps required as part of the build in the dockerfile and the beauty of this approach is that if those steps dont need re-running due to changes they simply wont be.

We can also use the image inheritence to our advantage by creating base images that contain and shared dependencies such as tooling or frameworks. And with careful ordering of the commands we can use the docker layers to ensure that work that has not triggered changes does not get done again.

In future I will show you other tricks such as how to use docker-compose as part of your development workflow and how to mount local code into a running container to have greated control of your local environments.

[The source code for this post is avilable on GitHub here](https://github.com/ScottGuymer/docker-build-for-windows-apps)