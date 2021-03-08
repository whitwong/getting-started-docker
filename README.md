# Docker Tutorial/Sample Application

Source: https://docs.docker.com/get-started/

## Tutorial Step-by-step
### <ins>Initial application setup
1. Copied `/app` directory from https://github.com/docker/getting-started/tree/master/app into personal repo.
1. Created `Dockerfile` into the `/app` directory. Added docker commands to `Dockerfile`.
1. cd into the `/app` directory and run the following command: `docker build -t getting-started-docker . `. This builds the docker container image using a tag name of `getting-started-docker` in the current directory (which is `/app`).
1. Start the app container! Use the following command: `docker run -dp 3000:3000 getting-started-docker`. The whole command says run the docker container `getting-started-docker` (tag name stated in the build command) using the flag designations. The `-dp` flags designate running the container in "detached" mode and create a "port" mapping from the host's port 3000 to the container's port 3000. The output after running this command is a sha (Secure Hash Algorithm) value.

### <ins>Update the application
1. After making changes, run the same build command. If you try to run/start the container immediately, you'll hit an error regarding a port already in use/allocated. This indicates that the old container is still running, and the error stems from the new container trying to use the host's port 3000 when only one process on the machine can listen to a specific port at one time. So how do we fix this?
1. Stop the old container and then remove it. Don't need to remove container after stopping it, but it's good practice.
1. Identify the running container you want to stop: `docker ps`
1. Once you have the Container ID, stop the container: `docker stop <container id>`
1. After the container stops (you can verify using `docker ps -a`), remove the container: `docker rm <container id>`
1. Bonus: You can stop and remove container at one time using the force flag `docker rm -f <container id>`

### <ins>Share the application
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

### <ins>Persist the DB
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

### <ins>Use bind mounts
Named volumes are great if we simply want to store data, as we don't have to worry about *where* the data is stored. With **bind mounts**, we control the exact mountpoint on the host. We can use this to persist data, but it's often used to provide addtional data into containers. When working on an application, we can use a bind mount to mount our source code into the container to let it see code changes, respond, and let us see the changes right away. For Node-based applications, nodemon is a great tool to watch for file changes and then restart the application. There are equivalent tools in most other languages and frameworks.

Bind mounts and named volumes are the two main types of volumes that come with the Docker engine. However, additional volume drivers are available to support other use cases (SFTP, Ceph, NetApp, S3, and more).

#### Start a dev-mode container
To run our container to support a development workflow, we will do the following:
  - Mount our source code into the container
  - Install all dependencies, including the "dev" dependencies
  - Start nodemon to watch for filesystem changes

Let's do it:
1. Make sure you don't have any previous `getting-started-docker` containers running. If you do, run `docker rm -f <container id>`.
1. cd into `/app` directory: `cd /app`. Run the following command:
```
docker run -dp 3000:3000 \
    -w /app -v "$(pwd):/app" \
    node:12-alpine \
    sh -c "yarn install && yarn run dev"
```
  - `-dp 3000:3000`: Run in detached (background) mode and create a port mapping
  - `-w /app`: Sets the "working directory" or the current directory that the command will run from
  - `-v "$(pwd):/app"`: Bind mount the current directory from the host in the container into the `/app` directory
  - `node:12-alpine`: The image to use. Note that this is the base image for our app from the Dockerfile
  - `sh -c "yarn install && yarn run dev"`: Start a shell use `sh` (alpine doesn't have bash) and run `yarn install` to install *all* dependencies and then run `yarn run dev`. If we look at `package.json`, we'll see that the `dev` script is starting `nodemon`.
3. You can watch the logs using `docker logs -f <container-id>`. You'll know you're ready to go when you see this:
```
docker logs -f <container-id>
$ nodemon src/index.js
[nodemon] 1.19.2
[nodemon] to restart at any time, enter `rs`
[nodemon] watching dir(s): *.*
[nodemon] starting `node src/index.js`
Using sqlite database at /etc/todos/todo.db
Listening on port 3000
```
When you're done watching the logs, exit out by hitting `Ctrl + C`.

4. Let's make a change to the app. In the `src/static/js/app.js` file, let's change the "Add Item" button to simply say "Add" (see line 109).
1. Simply refresh the page (or open it) and you should see the change reflected in the browser almost immediately. It might take a few seconds for the Node server to restart, so if you get an error try refreshing after a few seconds.
1. Feel free to make any other changes you'd like to make. When you're done, stop the container and build your new image using `docker build -t getting-started-docker .`.

Using bind mounts is *very* common for local development setups. The advantage is that the dev machine doesn't need to have all the build tools and environments installed. With a single `docker run` command, the dev environment is pulled and ready to go. 

### <ins>Multi container apps
Up to this point, we've been working with single container apps. But we now to add MySQL to the application stack. How do we want to set this up? Where will MySQL run? Install it in the same container or run it separately? In general, **each container should do one thing and do it well**. A few reasons to follow this:
- There's a good chance you'd have to scale APIs and frontends differently than databases
- Separate containers let you version and update versions in isolation
- While you may use a container for the database locally, you may want to use a managed service for the database in production. In this case, you don't want to ship your database engine with your app.
- Running multiple processes will require a process manager (the container only starts one process), which adds complexity to container startup/shutdown

#### Container networking
Containers by default run in isolation and don't knwo anythign about other processes or containers on the same machine. So...how do we allow one container to talk to another? The answer is **networking**. *If two containers are on the same network, they can talk to each other. If they aren't, they can't.*

#### Start MySQL
There are two ways to put a container on a network: 1) Assign it at start or 2) connect an existing container. For now, we will create the network first and attach the MySQL container at startup.

1. Create the network.
```docker network create todo-app```
1. Start a MySQL container and attach it to the network. We're also going to define a few environment variables that the database will use to initialize the database.
```
docker run -d \
  --network todo-app --network-alias mysql \
  -v todo-mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=todos \
  mysql:5.7
```
Notice that we used a volume named `todo-mysql-data` here and mounted it at `/var/lib/mysql`, which is where MySQL stores its data. However, we never ran a `docker volume create` command. Docker recognizes that we want to use a named volume and creates one automatically for us.

3. To confirm that we have the db up and running, connect to the db and verify it connects.
```
docker exec -it <mysql-container-id> mysql -p
```
When the password prompt comes up, type in **secret**. In the MySQL shell, list the databases and verify that the `todos` db exists. 
```
mysql> SHOW DATABASES;
```
You should see output that looks like this:
```
 +--------------------+
 | Database           |
 +--------------------+
 | information_schema |
 | mysql              |
 | performance_schema |
 | sys                |
 | todos              |
 +--------------------+
 5 rows in set (0.00 sec)
```
Sweet! We have our `todos` database and it's ready to use! 

#### Connect to MySQL
Now that we know MySQL is up and running, let's use it. But the question is...how? If we run another container on the same network, how do we find the container (remember each container has its own IP address)?

To figure it out, we're going to use the `nicolaka/netshoot` container, which ships with a *lot* of tools that are useful for troubleshooting or debugging networking issues.

1. Start a new container using the nicolaka/netshoot image. Make sure to connect it to the same network.
```
docker run -it --network todo-app nicolaka/netshoot
```
2. Inside the container, we're going to use the `dig` command, which is a useful DNS tool. We're going to look up the IP address for the hostmane `mesql`.
```
dig mysql
```
And you'll get an output like this:
```
; <<>> DiG 9.16.11 <<>> mysql
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 45781
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;mysql.				IN	A

;; ANSWER SECTION:
mysql.			600	IN	A	172.18.0.2

;; Query time: 4 msec
;; SERVER: 127.0.0.11#53(127.0.0.11)
;; WHEN: Mon Mar 08 03:57:58 UTC 2021
;; MSG SIZE  rcvd: 44
```
In the "ANSWER SECTION", you will see an `A` record for `mysql` that resolves to `172.18.0.2` (your IP address will most likely have a different value). While `mysql` isn't normally a valid hostname, Docker was able to resolve it to the IP address of the container that had that network alias (remember the `--network-alias` flag we used earlier?). What this means is...our app simple needs to connect to a host names `mysql` and it'll talk to the database! It doesn't get much simpler than that!

#### Run your app with MySQL
The todo app supports the setting of a few environment variables to specify MySQL connection settings. They are:
- `MYSQL_HOST` - the hostname for the running MySQL server
- `MYSQL_USER` - the username to use for the connection
- `MYSQL_PASSWORD` - the password to use for the connection
- `MYSQL_DB` - the database to use one connected

With all of that explained, let's start our dev-ready container!
1. We'll specify each of the environment variables above, as well as connect the container to our app network.
```
 docker run -dp 3000:3000 \
   -w /app -v "$(pwd):/app" \
   --network todo-app \
   -e MYSQL_HOST=mysql \
   -e MYSQL_USER=root \
   -e MYSQL_PASSWORD=secret \
   -e MYSQL_DB=todos \
   node:12-alpine \
   sh -c "yarn install && yarn run dev"
```
2. If we look at the logs for the container (`docker logs <container id>`), we should see a message indicating it's using the mysql database.
```
 # Previous log messages omitted
 $ nodemon src/index.js
 [nodemon] 1.19.2
 [nodemon] to restart at any time, enter `rs`
 [nodemon] watching dir(s): *.*
 [nodemon] starting `node src/index.js`
 Connected to mysql db at host mysql
 Listening on port 3000
```
3. Open the app in your browser and add a few items to your todo list.
1. Connect to the mysql database and prove that the items are being written to the db. Remember, the password is **secret**.
```
docker exec -it <mysql-container-id> mysql -p todos
```
And in the mysql shell, run the following:
```
mysql> select * from todo_items;
```
You should see your items listed in the table.




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
* Create a volume: `docker volume create <volume name>`
* List volumes: `docker volume ls`
* Remove volume: `docker volume rm <volume name>`
* Create a network: `docker network create <network name>`
* List networks: `docker network ls`