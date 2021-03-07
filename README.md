# Docker Tutorial/Sample Application

Source: https://docs.docker.com/get-started/

## Tutorial Step-by-step
### Initial application setup
1. Copied `/app` directory from https://github.com/docker/getting-started/tree/master/app into personal repo.
1. Created `Dockerfile` into the `/app` directory. Added docker commands to `Dockerfile`.
1. cd into the `/app` directory and run the following command: `docker build -t getting-started-docker . `. This builds the docker container image using a tag name of `getting-started-docker` in the current directory (which is `/app`).
1. Start the app container! Use the following command: `docker run -dp 3000:3000 getting-started-docker`. The whole command says run the docker container `getting-started-docker` (tag name stated in the build command) using the flag designations. The `-dp` flags designate running the container in "detached" mode and create a "port" mapping from the host's port 3000 to the container's port 3000. The output after running this command is a sha (Secure Hash Algorithm) value.

### Update the application
1. After making changes, run the same build command. If you try to run/start the container immediately, you'll hit an error regarding a port already in use/allocated. This indicates that the old container is still running, and the error stems from the new container trying to use the host's port 3000 when only one process on the machine can listen to a specific port at one time. So how do we fix this?
1. Stop the old container and then remove it. Don't need to remove container after stopping it, but it's good practice.
1. Identify the running container you want to stop: `docker ps`
1. Once you have the Container ID, stop the container: `docker stop <container id>`
1. After the container stops (you can verify using `docker ps -a`), remove the container: `docker rm <container id>`
1. Bonus: You can stop and remove container at one time using the force flag `docker rm -f <container id>`

### Share the application
#### Create a repo
1. Create a repository on [Docker Hub](hub.docker.com).
1. Name the repo (nameed this one `getting-started-docker`) and make sure visibility is `Public`.
1. Click the `Create` button.
#### Push the image
1. Notice the Docker command that hints at how to push a new tag to the repo: `docker push <dockerNamespace>/getting-started-docker:tagname`
1. If you try running that push command now, you'll get an error: 
```
Using default tag: latest
The push refers to repository [docker.io/<dockerNamespace>/getting-started-docker]
An image does not exist locally with the tag: <dockerNamespace>/getting-started-docker
```
3. Why did this happen? Well, the push command was looking for an image name `<dockerNamespace>/getting-started-docker` and didn't find one. If you list images `docker images`, you won't find it there either. To fix this, we need to "tag" our existing image to give it a different name. Let's do that.
1. Login to Docker Hub from the command line `docker login -u YOUR-USER-NAME`. Enter password when prompted.
1. Use the `docker tag` command to give the `getting-started-docker` image a new name: `docker tag getting-started-docker YOUR-USER-NAME/getting-started-docker`
1. Now try the push command again: `docker push YOUR-USER-NAME/getting-started`. It works! 🎉
#### Run the image on a new instance
1. Now that the image has been built and pushed into a registry, now it's time to try running the app on a brand new instance that has never seen this container image. Use Play with Docker to do this.
1. Open [Play with Docker](https://labs.play-with-docker.com/) in a new browser window.
1. Login using your Docker Hub account credentials. Select `Start`.
1. Click on the `ADD NEW INSTANCE` on the left side bar.
1. In the terminal, start the app! `docker run -dp 3000:3000 YOUR-USER-NAME/getting-started-docker`
1. Click on the 3000 badge when it comes up and this will open a new tab with your app running! If the 3000 badge doesn't show up, you can click on the "Open Port" button and type in 3000 manually.

### Persist the DB
#### Container Volumes
- Containers (what we currently know):
  - When a container runs, it uses the various layers from an image for its filesystem. Each container also gets its own "scratch space" to create/update/remove files. Any changes won't be seen in another container, even if they are using the same image.
  - Each container starts from the image defintion each time it starts.
  - While containers can create, update, and delete files, those changes are lost when the container is removed and all changes are isolated to that container. With volumes, we can change all of this.
- Volumes:
  - Volumes provide the ability to connect specific filesystem paths of the container back to the host machine.
  - If a directory in the container is mounted, changes in the directory are also seen on the host machine. Therefore if we mount the same directory across container restarts, we will see the same files.
  - There are 2 main types of volumes. We will eventually use both, but we'll start with **named volumes**.
#### Persist the todo data
By default (and at this stage in the example), the todo app stores its data in a SQLite Database at `/etc/todos/todo.db`. SQLite is a relational database that sotres all of the data in a single file. While this isn't the best for large-scale applications, this is great for small demos. We'll switch to a different db engine later.

As mentioned before, we're going to use a **named volume**. Think of a named volume as simply a bucket of data. Docker maintains the physical location on the disk and you only need to remember the name of volume. Every time you use the volume, Docker will make sure the correct data is provided.

1. Create a volume bby using the `docker volume create` command: `docker volume create todo-db`
1. Stop the todo app container once again in the Dashboard (or with `docker rm -f <container id>`), as it's still running without using the persistent volume.
1. Start the todo app container but add the `-v` flag to specify a volume mount. We will use the named volume and mount it to `/etc/todos`, which will capture all files created at the path. `docker run -dp 3000:3000 -v todo-db:/etc/todos getting-started-docker`
1. Once the container starts up, open the app and add a few items to your todo list.
1. Remove the container for the todo app. Use the Dashboard or `docker ps` to get the ID and then use `docker rm -f <container id>` to remove it.
1. Start a new container using the same command from above: `docker run -dp 3000:3000 -v todo-db:/etc/todos getting-started-docker`
1. Open the app. You should see you itmes still in your list!
1. Go ahead and remove the container when you're done checking out your list. Yay, we've learned how to persist data! 🎉

Note: While named, volumes and bind mounts (which we'll explore next) are the two main types of volumes supported by a default Docker engine installation, there are many volume driver plugins available to support NFS, STFP, NetApp, and more. THeis will be especially important once you start running containers on multiple hosts in a clustered environment with Swarm, Kubernetes, etc.
#### Dive into the volume
A frequently asked question is "Where is Docker *actually* storing my data when I use a named volume?" If you want to know, you can use the `docker volume inspect` command.
```
docker volume inspect todo-db
[
    {
        "CreatedAt": "2019-09-26T02:18:36Z",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/todo-db/_data",
        "Name": "todo-db",
        "Options": {},
        "Scope": "local"
    }
]
```
The `Mountpoint` is the actual location on the disk where the data is stored. Note that on most machines, you will need to have root access to this directory from the host. But that's where it is!



## Helpful commands
* Build docker image: `docker build -t <image name> .`
* Run the container: `docker run -dp 3000:3000 <image name>`
* List docker images on machine: `docker images` or `docker image ls`
* List running docker containers/processes: `docker ps`
* List all docker containers/processes (stopped and running): `docker ps -a`
* Start image/container: `docker start <image name>` or `docker start <container id>`
* Stop image/container: `docker stop <image name>` or `docker stop <container id>`
* Delete container: `docker rm <container id>`
* Stop and remove container in one command: `docker rm -f <container id>`