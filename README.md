# Docker Cheat Sheet

Consise list of commands and notes that explain docker in my way.

## Docker CLI

It's rare to use the cli because Docker desktop is great, but these commands can be helpful
to learn the basics. and verify deployments later.

```bash
docker run node
```

By default a container is isolated from the surrounding environment.  The above command creates a container but it will appear
that nothing happened afterwards because it is isolated.  A new container will be visible in dockerdesktop but it will not
be running.


**List RUNNING Docker Processes on System**

```bash
docker ps
```
**List ALL Docker Processes on System**

This will list all proccesses on system regardless of status, add the `-a` flag:

```bash List All Processes
docker ps -a
```

If you want to work inside of the newly created container then you can  specify '-it' flag for an interactive terminal
inside of the container. (note that each time you specify run command you will create  anew container)

```bash
docker run -it node
```

## Dockerfile

More often when working with docker you'll use files to control the logic rather than raw cli commands. The most
basic approach is through the use of a Docker File.  When you write a Dockerfile you are creating a custom
image for Docker to build from, and you will always be building from a base image.

Dockerfile contains instructions for Docker to make our own image.

To get started with a Dockerfile you will specify a baseImage to build from, these are known keywords that resolve
to images.  If you don't have the specific image installed on your system then it will be downloaded automatically
from DockerHub.

In the below example we are using the pre-build *node* image as a baseImage.

```Dockerfile
FROM node
```

It's important to understand that an image has it's *Own* filesytem with directories etc, if you want files from
your project to be moved into the image's filesystem for use then you will need to use the `COPY` command.

```Dockerfile
COPY path1 path2
```

Path1 starts at the same location as the Dockerfile

Path2 starts at the root of the image

The most basic command you could run using `COPY` is to copy all files from the project root into the image root, you specify
this with a simple `.`, again this copies ALL project files into the root of the image.

```Dockerfile
COPY . .
```

Generally when it comes to the image you want to leave the root path alone so its common to specify a custom path name of your choosing
for the files to live.

```Dockerfile
COPY . /app
```

You can also run commands in the image as you would with a normal system, but by default commands will run in 
the working directory and not in the image.  To workaround this you can specify where Docker should execute
commands via the `WORKDIR` command.

```Dockerfile
RUN npm install
```

The above command alone would run on our local system.

```Dockerfile
WORKDIR /app

RUN npm install
```

The combo of the two commands above means that the npm install will now run in the /app directory of the image.

`RUN` vs `CMD`.  An important concept with images is that they are *Instructions* for building containers and
you would not want to `RUN` commands to start the service as part of the instructions.

SO `RUN node server.js` would be incorrect because our image would attempt to run the server as part of building
the container which is not what we want.  Instead we want to use the `CMD` command to specify the commands that
should be used when something starts our container which uses an array syntax.

```Dockerfile
CMD ['node' 'server.js']
``` 

If you don't specify a `CMD` as part of your image, the `CMD` of the base image will be executed as default.


## Networking

You need to keep in mind that just because a container is running a server on port 80, it does not mean that
you have access to it. Containers are isolated in nature so you need to explicitly expose a port if you want
it to be ableto connect to it from the outside. For that you need to use the `EXPOSE` command to specify port:

```Dockerfile
EXPOSE 80
```

So combining all of these concepts we are able to create the *Image* for a container that runs a node server application:

```Dockerfile
FROM node

WORKDIR /app

COPY . /app

RUN npm install

EXPOSE 80

CMD ["node", "server.js"]
```

## Build Command

Now that we've created a Dockerfile which contains instructions on how to build an image we want to build it. `BUILD` vs `RUN` are
very different commands, we used `RUN` above to startup a node image that was already created for us.  But our custom image has not
been built yet, so what we'd want to do is *build* our image using:

```bash
docker build .
```

This command tells Docker to *build an image* using a Dockerfile. The `.` in this case tells docker where the Dockerfile is so in this
case it means we're running `docker build` from the same directory that our docker file exists in.  If your build command is successful
there will be a bunch of output, the most important piece being the image ID.  Using the image id you can actually run the container
now.

```bash
docker run 283472734
```

This will start a container based on a built image, and you'll notice that it hangs the terminal because the node process from our image does not stop, this
is exactly what we expect.

**NOTE** `docker run` will ALWAYS start a new container, if you want to restart a container that was previously stopped this not the command you
want to use.

## EXPOSE BS, and publishing containers

It's a good idea to add an `EXPOSE` command because it serves as good documentation, but it doesn't actually expose that port. In fact
you could leave it out entirely because the acutal port forwarding is controlled when you publish a container while running it. So for
our example you can expose container's port 80 and map it to the local system's port 3000:

```bash
docker run -p 3000:80 283472734
```

## Relationship between images and containers

images are immutable. If you make changes to your application code and then restart the container you will notice no changes,
this is because the image was already built.  So if you make any changes to your application code you need to rebuild the image
if you want to see those changes. This is what makes containers so secure.


# Image Layers

Each instruction in a Dockerfile represents a layer in an image.  When you rebuild an image any layers that did not change will
be run from cache rather than rebuilding.  This makes things much more efficient and points out that Docker is smart enough to 
make decisions on it's own about how to speed up build times.

Note that as soon as a layer is determined to require a rebuild, ALL SUBSEQUENT layers will be rebuilt ignoring cache.

Understanding this concept points out something that can be optimized in our previous Dockerfile.  Currently we copy all files
over and then run `npm install`.  Because of the layer concept, this means that any change to our project will cause a rebuilt
from the COPY step and ALL SUBSEQUENT layers will run without cache.  The bad side effect here is that we will be running `npm install`
even if we don't need to do so, that means the better practice is to extract the package.json step and install above our project code
like so:

```Dockerfile
FROM node

WORKDIR /app

COPY package.json /app

RUN npm install

COPY . /app

EXPOSE 80

CMD ["node", "server.js"]
```

Now this means that when building the image, changes to our application code will not automatically cause an npm install because
that step exists on a layer above.

## Restart a stopped container

```bash
Docker start 283472734
```

This brings the container back up into status: RUNNING. And it does so in "detached" mode so your terminal won't be blocked with
logs from the container.  More on that below.

## Attached/Detatched Containers

Attached (the default with `docker run`) means you are listening to the output of that container from the same process (terminal), things
like logs from an express server etc would be visible here.

Detached (the default wiht `docker start`) means that the process runs in the background and you are not listening to the output of the
container in the same terminal. This allows for you to continue using your terminal as you normally would.

## Access logs from detatched containers

If you had started a container in detached mode but were cuious about the logs, you can actually pull up all logs from a detached container after
 the fact. And, you will have the all historical logs from the container as well. There is the `-f` flag if you only want logs from that point on.

## Removing containers

**NOTE**: You cannot remove a running container, stop the container first.

```bash
docker rm 283472734
```

remove images

```bash
docker rmi node
```

**NOTE**:  this will only work when the image is not used by ANY container (stopped or running)


## Remove stopped containers automatically

In a real-world setting if you were developing with docker you could need to update your image and hence your container frequently, this means a lot
of obsolete images will quickly mount up.  To ensure these are automatically deleted once they stop, add `--rm` flag to the run command, ex:

```bash
docker run -p 3000:80 -d --rm 283472734
```

remove all unused images with *prune*

## Definitions

Container
    - An isolated unit of software which is based on an image.