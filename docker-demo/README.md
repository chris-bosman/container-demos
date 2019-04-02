# Docker Demo #

Firstly, follow the instructions [here](https://www.docker.com/community-edition#/download) to download Docker for your OS. You will have to create an account for Docker to run the software.

Assuming we have Docker installed correctly, let's do some stuff.

## Getting Started ##

First, let's clone this repo.
`git clone https://github.com/chris-bosman/docker-demo.git`

Second, let's look at images Docker has available for NodeJS:

```console
$ docker search node
NAME                                   DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
node                                   Node.js is a JavaScript-based platform for s…   5909                [OK]
mhart/alpine-node                      Minimal Node.js built on Alpine Linux           368
mongo-express                          Web-based MongoDB admin interface, written w…   274                 [OK]
nodered/node-red-docker                Node-RED Docker images.                         171                                     [OK]
selenium/node-chrome                                                                   162                                     [OK]
...
```

The output above is everything from Docker's Container Image Registry that is tagged with the word "node". Let's pull a couple.

```console
$ docker pull node:latest
$ docker pull node:9
```

We've pulled two NodeJS images. One is the "latest" one, which is the latest committed image from the provider, and one which is tagged with the verison "9". You can go directly to the Docker [Hub](https://hub.docker.com) to search for images that way as well.

Okay let's look at the images we've downloaded:

```console
$ docker image ls
REPOSITORY                                   TAG                 IMAGE ID            CREATED             SIZE
node                                         latest              xxxxxxxxxxxx        2 minutes ago       674MB
node                                         9                   xxxxxxxxxxxx        2 minutes ago       673MB
```

## Running Containers ##

Let's boot a container with the "9" tag.

```console
$ docker run -d -t node:9
dc8eba68480910bedb4b2dcdfedfe126afce1d2a37d5c5244ec242a5b198a5a0
```

The `-d` flag here runs the container in "detached" mode, which means it'll run in the background and let us keep use of our terminal window. Without it, we would have to open up a new terminal to run commands on the container. The `-t` flag gives the container a psuedo-TTY, and without it containers that do not have a start-up command (which is most base containers) will kill themselves off upon starting up without error.

Alright, now we're going to run some commands on the Docker image for version 9 that we've installed. These commands will get the Container Name, variablize it, and get the running version of Node.

```console
$ CONTAINER_NAME=$(docker ps | awk '{if(NR>1) print $NF}')
$ docker exec $CONTAINER_NAME node -v
```

Cool. Let's do something more substantial. Let's run an Express-based Hello World app on our running container.

```console
$ docker exec $CONTAINER_NAME mkdir /app/
$ docker cp ./expressjs/ $CONTAINER_NAME:/app/
$ docker exec $CONTAINER_NAME bash -c "cd /app/expressjs && npm i"
$ docker exec $CONTAINER_NAME bash -c "cd /app/expressjs && npm start"

> docker_web_app@1.0.0 start /app
> node hello-world.js

Running on http://localhost:3000
```

Now, let's hit http://localhost:3000 and see our Hello World app!

![ErrorPic](https://raw.githubusercontent.com/chris-bosman/Public-Scripts/master/error-pic.jpg)

Oh no. So, what happened?

Well, in using separate namespaces, Docker actually segregates the network traffic between the container and our local computer. This means we have to explicitly tell Docker to serve up the site on the 3000 port so our local machine will recognize it. Also, we can't do this to a running container, so we'll have to kill our container and re-start it with the proper commands.

```console
$ docker kill $CONTAINER_NAME
$ docker run -d -t -p 3000:3000 node:9
2b845beffed82e62ace124212acef98253cac3a8c96dcaaf57644829ec8029fc
$ CONTAINER_NAME=$(docker ps | awk '{if(NR>1) print $NF}')
$ docker exec $CONTAINER_NAME mkdir /app/
$ docker cp ./expressjs/ $CONTAINER_NAME:/app/
$ docker exec $CONTAINER_NAME bash -c "cd /app/expressjs && npm i"
npm notice created a lockfile as package-lock.json. You should commit this file.
npm WARN docker_web_app@1.0.0 No repository field.
npm WARN docker_web_app@1.0.0 No license field.

added 50 packages in 2.413s
$ docker exec $CONTAINER_NAME bash -c "cd /app/expressjs && npm start"

> docker_web_app@1.0.0 start /app
> node hello-world.js

Running on http://localhost:3000
```

Let's go to https://localhost:3000 again.

![hello-world-image](https://raw.githubusercontent.com/chris-bosman/Public-Scripts/master/hello-world.jpg)

That's more like it!

## Interactive Mode ##

First, let's kill our existing running container.

`docker kill $CONTAINER_NAME`

Now, you can maybe see the issues with the above approach. Firstly, having to constantly pipe commands through `docker exec` is cumbersome and can be error-prone. Secondly, because many configuration changes to Docker containers require the full death & birth of a new container (like our port issue), and also because containers are stateless and do not retain changes, we had to re-run commands multiple times to get our container up and running properly, which is a less than ideal scenario.

The workaround to the first issue is simple. Instead of running containers in detached mode, (`-d`) we can run them in interactive mode and call the shell with a command like this: `docker run -i -t node:9 bash`. This grants your running terminal the context of the docker container that you're starting. You can also use `docker exec -it $CONTAINER_NAME bash` to do the same thing to a container you started in detached mode. In either case, the `exit` command will take you back to your local machine's shell.

When within the container's shell, you can treat it like your own environment, running commands, installing software, etc. If you need to copy items from your local machine to the container, you can either `exit` or open a new terminal and then run the `docker cp` command.

## Dockerfiles ##

For the second issue, it's time to introduce you to Dockerfiles. Dockerfiles are specialized files that script out the commands you want to run on a container in a declarative language. For example, let's consider the commands we had to run to start our hello-world app:

    docker run -d -t -p 3000:3000 node:9
    CONTAINER_NAME=$(docker ps | awk '{if(NR>1) print $NF}')
    docker exec $CONTAINER_NAME mkdir /app/
    docker cp ./expressjs/ $CONTAINER_NAME:/app/
    docker exec $CONTAINER_NAME bash -c "cd /app/expressjs && npm i"
    docker exec $CONTAINER_NAME bash -c "cd /app/expressjs && npm start"

In a Dockerfile, that work can be declared like this:

    FROM node:9
    COPY ./expressjs/ /app/
    WORKDIR app/expressjs
    RUN npm i
    EXPOSE 3000
    CMD npm start

That Dockerfile can then be built into a custom image, and tagged with its own name with the command `docker build . -t node-hello-world`. The `-t` here is the "tag" flag. This flag tags the container image with a name that you can then use to call it again in other commands.

```console
$ docker build . -t node-hello-world
Sending build context to Docker daemon  39.94kB
Step 1/7 : FROM node:9
9: Pulling from library/node
d660b1f15b9b: Already exists
46dde23c37b3: Already exists
6ebaeb074589: Already exists
e7428f935583: Already exists
eda527043444: Already exists
f3088daa8887: Already exists
1ded38ff7fdc: Pull complete
da44c9274f48: Pull complete
Digest: sha256:cddc729ef8326f7e8966c246ba2e87bad4c15365494ff3d681fa6f022cdab041
Status: Downloaded newer image for node:9
 ---> 08a8c8089ab1
Step 2/7 : COPY ./expressjs/ /app/
 ---> 5be9f073224e
Step 3/7 : WORKDIR app/expressjs
 ---> Running in 79433a3c82e7
Removing intermediate container 79433a3c82e7
 ---> 3f636a57e3fb
Step 4/7 : RUN npm i
 ---> Running in 88b34efe5a35
npm notice created a lockfile as package-lock.json. You should commit this file.
npm WARN docker_web_app@1.0.0 No repository field.
npm WARN docker_web_app@1.0.0 No license field.

added 50 packages in 2.385s
Removing intermediate container 88b34efe5a35
 ---> 85cd2651e1bf
Step 5/7 : EXPOSE 3000
 ---> Running in e4a79cbe45fd
Removing intermediate container e4a79cbe45fd
 ---> fed0cdfcd247
Step 6/7 : ENTRYPOINT [ "npm" ]
 ---> Running in 2af32ac19250
Removing intermediate container 2af32ac19250
 ---> cf7055501a6f
Step 7/7 : CMD [ "start" ]
 ---> Running in a648bac87a29
Removing intermediate container a648bac87a29
 ---> 654d061652af
Successfully built 654d061652af
Successfully tagged node-hello-world:latest
```

And then run with `docker run -d -p 3000:3000 node-hello-world`. (Note two things: 1. How we're using the name we tagged in the previous command to call the correct docker image, and 2. That we still have to explicitly command docker to give the local machine access to port 3000, even with declaring it in our Dockerfile.)

If you run `docker image ls` you can also now see the image we built with our Dockerfile.

```console
$ docker image ls
REPOSITORY                                   TAG                 IMAGE ID            CREATED             SIZE
node-hello-world                             latest              654d061652af        8 minutes ago       676MB
```

To cleanup, we should kill our image again.

```console
$ CONTAINER_NAME=$(docker ps | awk '{if(NR>1) print $NF}')
$ docker kill $CONTAINER_NAME
```

## Container Registries & Managing Docker Images ##

This is great if we just want to build, modify, and run images locally, but what about when we want to provide them to others? That's where Container Registries come in. For the purposes of this demo, you should establish your own repository. You can get one free private repository and unlimited free public repositories from Docker by logging into the website, going to your profile, and selecting Create -> Create Repository.

Logging into your own private Docker Repository and logging into the Azure Container Registry utilize different commands.

To log into a private Docker repository, run:

```console
$ docker login https://registry.hub.docker.com/r/[docker-username]/node-hello-world/
Username: [docker-username]
Password:
Login Succeeded
```

Now, how do we prep our Dockerfile-created images and get them to our container registry? By using tags. By prepending an image name with the name of a Container Registry, we're telling Docker that we want to register that image with our Registry. Do this by running `docker tag node-hello-world [docker-username]/node-hello-world`.

Please note, by default Docker adds an implicit "latest" sub-tag to any images it creates, but Docker best practices also suggest that you use sub-tags to declare explicit versions of your images. You add a sub-tag by adding a colon after the image name, with the name of the sub-tag following, I.E.: `docker tag node-hello-world [docker-username]/node-hello-world:1.0` Your sub-tags can be as verbose as you'd like and, additionally, you can tag the same image with multiple sub-tags.

Now that we've logged into our Container Registry and tagged our image with its URL, we can push this image to our Registry with `docker push [docker-username]/node-hello-world:[sub-tag]`.

```console
$ docker push [docker-username]/node-hello-world:latest
The push refers to repository [[docker-username]/node-hello-world]
213e1fcf7834: Pushed
5bccc894c47b: Pushed
d33bfb772e63: Pushed
71521673e105: Pushed
7695686f75c0: Pushed
e492023cc4f9: Mounted from [other image]
cbda574aa37a: Mounted from [other image]
8451f9fe0016: Mounted from [other image]
858cd8541f7e: Mounted from [other image]
a42d312a03bb: Mounted from [other image]
dd1eb1fd7e08: Mounted from [other image]
latest: digest: sha256:94f7494203d026202290ef8efb12dd78505704974c8ededea4897be84fac94cd size: 2632
```

Some interesting things to note here. One, this is a good illustration of the Docker concept of layers. Each Docker image is broken down into sets of smaller layers, and those layers are given a UUID. These layers are shared among **all appropriate images**. For example, all images that use the same base OS image (let's say, Debian) will share some layers. Even if thousands of Debian images exist in Docker, or in a Container Registry, the layer only exists once and is simply referenced thousands of times. You can see above that thise image was broken down into 11 layers. Docker does this layering itself; it requires no input or configuration on the user end.

Upon pushes and pulls, those layers are evaluated both locally and against the registry. When pulling an image, your local machine will reuse any layers that already exist and only download layers that have changed. Similarly, when pushing a new image, your machine will only upload layers that have changed from what already exists in the Registry. As you can also see above, if images in your local repository share layers, Docker is smart enough to use existing layers even from different images upon pushes and pulls.

And when you are pushing an image with the exact same layers but just a different sub-tag?

```console
$ docker push [docker-username]/node-hello-world:1.0
The push refers to repository [[docker-username]/node-hello-world]
213e1fcf7834: Layer already exists
5bccc894c47b: Layer already exists
d33bfb772e63: Layer already exists
71521673e105: Layer already exists
7695686f75c0: Layer already exists
e492023cc4f9: Layer already exists
cbda574aa37a: Layer already exists
8451f9fe0016: Layer already exists
858cd8541f7e: Layer already exists
a42d312a03bb: Layer already exists
dd1eb1fd7e08: Layer already exists
1.0: digest: sha256:94f7494203d026202290ef8efb12dd78505704974c8ededea4897be84fac94cd size: 2632
```

Docker evaluates that the layers all already exist in the registry and simply adds a tag to those layers. This functionality keeps storage costs down and allows images to be downloaded, uploaded, and built with much less overhead.

## Using Private Containers ##

Okay, so now we have our image in our private Container Registry. What does that gain us?

Well, now we can pull our image from the registry and use it, pre-built, with the Express instance already running, with one command!

```console
$ docker run -d -p 3000:3000 [docker-username]/node-hello-world
8f08d11321275bd535b7a9d10e154e9e9f1300cd96b7d63c54501d7463e12736
```

Navigate to http://localhost:3000 again and see that your Hello World is running. Huzzah!

## Cleanup ##

Let's ensure we don't leave our container running.

`docker kill $CONTAINER_NAME`