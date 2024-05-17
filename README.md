# Docker Cheat Sheet

Consise list of commands and notes that explain docker in my way.

## Docker CLI

It's rare to use the cli because Docker desktop is great, but these commands can be helpful
to learn the basics. and verify deployments later.

```bash
docker run node
```

By default a container is isolated from the surrounding environment. The above command creates a container but it will appear
that nothing happened afterwards because it is isolated. A new container will be visible in dockerdesktop but it will not
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

If you want to work inside of the newly created container then you can specify '-it' flag for an interactive terminal
inside of the container. (note that each time you specify run command you will create anew container)

```bash
docker run -it node
```

## Dockerfile

More often when working with docker you'll use files to control the logic rather than raw cli commands. The most
basic approach is through the use of a Docker File. When you write a Dockerfile you are creating a custom
image for Docker to build from, and you will always be building from a base image.

Dockerfile contains instructions for Docker to make our own image.

To get started with a Dockerfile you will specify a baseImage to build from, these are known keywords that resolve
to images. If you don't have the specific image installed on your system then it will be downloaded automatically
from DockerHub.

In the below example we are using the pre-build _node_ image as a baseImage.

```Dockerfile
FROM node
```

It's important to understand that an image has it's _Own_ filesytem with directories etc, if you want files from
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
the working directory and not in the image. To workaround this you can specify where Docker should execute
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

`RUN` vs `CMD`. An important concept with images is that they are _Instructions_ for building containers and
you would not want to `RUN` commands to start the service as part of the instructions.

SO `RUN node server.js` would be incorrect because our image would attempt to run the server as part of building
the container which is not what we want. Instead we want to use the `CMD` command to specify the commands that
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

So combining all of these concepts we are able to create the _Image_ for a container that runs a node server application:

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
very different commands, we used `RUN` above to startup a node image that was already created for us. But our custom image has not
been built yet, so what we'd want to do is _build_ our image using:

```bash
docker build .
```

This command tells Docker to _build an image_ using a Dockerfile. The `.` in this case tells docker where the Dockerfile is so in this
case it means we're running `docker build` from the same directory that our docker file exists in. If your build command is successful
there will be a bunch of output, the most important piece being the image ID. Using the image id you can actually run the container
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
this is because the image was already built. So if you make any changes to your application code you need to rebuild the image
if you want to see those changes. This is what makes containers so secure.

# Image Layers

Each instruction in a Dockerfile represents a layer in an image. When you rebuild an image any layers that did not change will
be run from cache rather than rebuilding. This makes things much more efficient and points out that Docker is smart enough to
make decisions on it's own about how to speed up build times.

Note that as soon as a layer is determined to require a rebuild, ALL SUBSEQUENT layers will be rebuilt ignoring cache.

Understanding this concept points out something that can be optimized in our previous Dockerfile. Currently we copy all files
over and then run `npm install`. Because of the layer concept, this means that any change to our project will cause a rebuilt
from the COPY step and ALL SUBSEQUENT layers will run without cache. The bad side effect here is that we will be running `npm install`
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
logs from the container. More on that below.

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

**NOTE**: this will only work when the image is not used by ANY container (stopped or running)

## Remove stopped containers automatically

In a real-world setting if you were developing with docker you could need to update your image and hence your container frequently, this means a lot
of obsolete images will quickly mount up. To ensure these are automatically deleted once they stop, add `--rm` flag to the run command, ex:

```bash
docker run -p 3000:80 -d --rm 283472734
```

remove all unused images with _prune_

## Inspect Images

```bash
docker inspect image 283472734
```

This will show a huge readout with details about the image. Lots of basic information but one thing worth
looking at is the **OS**, all baseImages have some sort of underlying OS to operate. For example the node
image has the node runtime but it requires an operating system to run on, so in that case the OS is _Linux_

You can also look at all the layers in your image which will reveal that you also take the layers
from any base image you started from. this is how caches can be cleared when a base image is out of daate.

You will rarely use this command but its interesting.

## Copying to/from container

Docker supplies a method to copy files into or out of a running container which is pretty cool.

```bash
docker cp <local glob> <container path to copy to>
```

This will also be seldom used, but good to know about.

## Data Management

Our application code is technically data but it can't be modified after the imaage
is built, this is actually a good thing when you consider security. But how do we
manage data that we want to update and maintain?

Understand that images are Read-Only but containers are read/write (demonstrated with
the `cp` command). container data only lasts the duration of the container though
and is then lost, how can we manage permanent data? (Volumes)

### Data problem

A container maintains it's own filesystem etc and you can write and transfer files
to it exactly how you would want to. Even if you stop & restart the continer the written
files will still be there. The problem happens when you need to rebuild an image, these files
will not be there because the data is not part of the image so here lies the problem.

## Volumes

Volumes are folders on your HOST machine which are mounted (made available) to containers. If a
container is shut down the volume lives on, it has a totally separate lifecycle to the container.

It's possible to create anonymous volumes from within a Dockerfile but this is not what we need for persisted
storage. instead you need to reference a _named_ volume when you start a container. named volumes will stay
up even after a container is shut down.

```bash
-v volumename:/container-path
```

## Bind Mounts

Great for persistent, editable data.

It can solve the problem of us constantly having to rebuild our image and container whenever we make changes.
You define bind-mounts from inside the terminal rather than the Dockerfile, since it haas nothing to do with
the image but instead the container.

you do so by sharing the absolute path to you project folder followed by `:/app`

example:

```bash
-v "/Users/vinnie/path-to-project:/app"
```

This maps our entire project as a volume into the /app folder of the container

You could also use a shortcut rather than writing the full absolute path

```bash
-v $(pwd):/app
```

this accomplishes the same result as the command above.

I'm not exactly sure what the advantage of this is yet since usually we're not building the container for
local development in my experience, but that might be different outside of webapps.

In video: Combineing & Merging different volumes Max demos how to overwrite parts of the container's files
with your local but also put node modules in the container only... could be useful but not a core concept I'm
interested in right now.

One cool side-effect is that it makes it so node_modules never get built to your machine, could be handy if you
have a lot of repos building up because you could just destroy the container and the whole node_modules folder with it.

# Volumes & BindMount summary

1. anonymous volume that gets deleted when the container does (useful for node_modules)

```bash
docker run -v /app/data
```

2. named volume, useful for saving data permanently by binding a folder on the host machine
   to your own.

```bash
docker run -v dataa:/app/data
```

3. bind mount: overwrite container content with host machine content (application code)

```bash
docker run -v /path/to/code:/app/code
```

## Reading/Writing Volumes

By default volumes are read/write, but you can set them up to be
RO if you want. This could be the case for a bind-mount with your application
code since it shouldn't be modified from the container. just end the path with `:ro`

## Copy vs BindMounts

bindmounts are not a replacement for COPY in the dockerfile which does sort of the same
thing. bindmounts will not be used in production because source code is not changing on
the fly, so you need to make sure that your COPY command is still present in the Dockerfile.

## Dockerignore files

They work just like the ones from git and take format:

`.dockerignore`

You would likely want to still place `node_modules` here as well because you
wouldn't want your local modules to overwrite the contaainer as well.

There are other examples as well like the .git folder, or even the Dockerfile
itself lol.

## ARGuments & ENVironment Variables

ARG: available inside of Dockerfile, NOT accessible in CMD or application code. provided
by: `--build-arg` during `docker build`

ENV: available in Dockerfile just like ARG, but also available inside of the application
code.

To set an evnironment variable in the container via the Dockerfile use:

```Dockerfile
ENV PORT 80

EXPOSE $PORT
```

now in node the value is accessilbe as:

```js
process.env.PORT // 80
```

when you build the container you would then use:

```bash
--env PORT=8000
```

This would overwrite the default value from the Dockerfile now.

You can also just forward your whole .env file of your project
when using docker build with:

```bash
--env-file ./.env
```

The relationship to args is a little strange, but to creaate truly default & optional
you would use a combo of args and envs. ARG cannot be used in the application code
so you would need to map it to an ENV if you wanted to use it there.

I guess the advantage of args is you wouldn't need to set any new values in a file. Best
to put them underneath npm install because of the layers concept.

## Definitions

Container - An isolated unit of software which is based on an image.
