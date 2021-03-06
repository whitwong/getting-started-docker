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
2. Inside the container, we're going to use the `dig` command, which is a useful DNS tool. We're going to look up the IP address for the hostname `mysql`.
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
In the "ANSWER SECTION", you will see an `A` record for `mysql` that resolves to `172.18.0.2` (your IP address will most likely have a different value). While `mysql` isn't normally a valid hostname, Docker was able to resolve it to the IP address of the container that had that network alias (remember the `--network-alias` flag we used earlier?). What this means is...our app simply needs to connect to a host named `mysql` and it'll talk to the database! It doesn't get much simpler than that!

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

### <ins>Use Docker Compose
Docker Compose is a tool that was developed to help define and share multi-container applications. With Compose, we can create a YAML file to define the services and with a single command, can spin everything up or tear it all down.

The *big* advantage of using Compose is you can define your application stack in a file, keep it at the root of your project repo (it's now version controlled), and easily enable someone else to contribute to your project. Someone would only need to clone your repo and start the compose app.

#### Install Docker Compose
If you install Docker Desktop/Toolbox for either Windows or Mac, you already have Docker Compose. Play-with-Docker instances already have Docker Compose installed as well. If you are on a Linus machine, you will need to install Docker Compose. After installation, you should be able to run the following and see version information.
```
docker-compose version
```

#### Create the Compose file
1. At the root of the app project, create a file named `docker-compose.yml`.
1. In the compose file, we'll start off by defining the schema version. In most cases, it's best to use the latest supported version. You can look at the [Compose file reference](https://docs.docker.com/compose/compose-file/) for the current schema versions and the compatibility matrix. 
```
version: "3.7"
```
3. Next we'll define the list of services (or containers) we want to run as part of our application.
```
version: "3.7"

services: 
```
And now we'll start migrating a service at a time into the compose file.

#### Define the app service
To remember, this was the command we were using to define our app container.
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
1. First, let's define the service entry and the image for the container. We can pick any name for the service, in this case we'll name it `app`. The name will automatically become a network alias, which will be useful when defining our MySQL service.
```
 version: "3.7"

 services:
   app:
     image: node:12-alpine
```
2. Typically, you will see the command close to the `image` definition, although there is no requirement on ordering. So, let's go ahead and move that into our file.
```
 version: "3.7"

 services:
   app:
     image: node:12-alpine
     command: sh -c "yarn install && yarn run dev"
```
3. Let's migrate the `-p 3000:3000` part of the command by defining the `ports` for the service. We will use the [short syntax](https://docs.docker.com/compose/compose-file/compose-file-v3/#short-syntax-1) here, but there is also a more verbose [long syntax](https://docs.docker.com/compose/compose-file/compose-file-v3/#long-syntax-1) available.
```
 version: "3.7"

 services:
   app:
     image: node:12-alpine
     command: sh -c "yarn install && yarn run dev"
     ports:
       - 3000:3000
```
4. Next, we'll migrate both the working directory (`-w /app`) and the volume mapping (`-v "$(pwd):/app"`) by using the `working_dir` and `volumes` definitions. Volumes also has a short and long syntax. One advantage of Docker Compose volume definitions is we can use relative paths from the current directory.
```
 version: "3.7"

 services:
   app:
     image: node:12-alpine
     command: sh -c "yarn install && yarn run dev"
     ports:
       - 3000:3000
     working_dir: /app
     volumes:
       - ./:/app
```
5. Finally, we need to migrate the environment variable definitions using the `environment` key.
```
 version: "3.7"

 services:
   app:
     image: node:12-alpine
     command: sh -c "yarn install && yarn run dev"
     ports:
       - 3000:3000
     working_dir: /app
     volumes:
       - ./:/app
     environment:
       MYSQL_HOST: mysql
       MYSQL_USER: root
       MYSQL_PASSWORD: secret
       MYSQL_DB: todos
```

#### Define the MySQL service
Now it's time to define the MySQL service. The command that we used for that container was the following:
```
docker run -d \
  --network todo-app --network-alias mysql \
  -v todo-mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=todos \
  mysql:5.7
```
1. We will first define the new service and name it `mysql` so it automatically gets the network alias. We'll go ahead and specify the image to use as well.
```
 version: "3.7"

 services:
   app:
     # The app service definition
   mysql:
     image: mysql:5.7
```
2. Next we'll define the volume mapping. When we ran the container with `docker run`, the named volume was created automatically. However, that doesn't happen when running with Compose. We need to define the volume in the top-level `volumes:` section and then specify the mountpoint in the service config. By simple providing only the volume name, the default options are used. There are [many more options available](https://docs.docker.com/compose/compose-file/compose-file-v3/#volume-configuration-reference) though.
```
 version: "3.7"

 services:
   app:
     # The app service definition
   mysql:
     image: mysql:5.7
     volumes:
       - todo-mysql-data:/var/lib/mysql
    
 volumes:
   todo-mysql-data:
```
3. Finally, we only need to specify the environment variables.
```
 version: "3.7"

 services:
   app:
     # The app service definition
   mysql:
     image: mysql:5.7
     volumes:
       - todo-mysql-data:/var/lib/mysql
     environment: 
       MYSQL_ROOT_PASSWORD: secret
       MYSQL_DATABASE: todos
    
 volumes:
   todo-mysql-data:
```
At this point, our complete `docker-compose.yml` should look like this:
```
version: "3.7"

services:
  app:
    image: node:12-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 3000:3000
    working_dir: /app
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: secret
      MYSQL_DB: todos

  mysql:
    image: mysql:5.7
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment: 
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos

volumes:
  todo-mysql-data:
```

#### Run the application stack
Now that we have our `docker-compose.yml` file, we can start it up!
1. Make sure no other copies of the app/db are running first (`docker ps` and `docker rm -f <ids>`).
1. Start up the application stack using the `docker-compose up` command. We'll add the `-d` flag to run everything in the background.
```
docker-compose up -d
```
When we run this, we should see output like this:
```
 Creating network "app_default" with the default driver
 Creating volume "app_todo-mysql-data" with default driver
 Creating app_app_1   ... done
 Creating app_mysql_1 ... done
```
You'll notice that the volume was created as well as a network! By default, Docker Compose automatically creates a network specifically for the application stack (which is why we didn't define one in the compose file).

3. Let's look at the logs use the `docker-compose logs -f` command. You'll see the logs from each of the services interleaved into a single stream. This is incredibly useful when you want to watch for timing-related issues. The `-f` flag "follows" the log, so will give you live output as it's generated.

If you don't already, you'll see output that looks like this:
```
 mysql_1  | 2019-10-03T03:07:16.083639Z 0 [Note] mysqld: ready for connections.
 mysql_1  | Version: '5.7.27'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server (GPL)
 app_1    | Connected to mysql db at host mysql
 app_1    | Listening on port 3000
```
The service name is displayed at the beginning of the line (often colored) to help distinguish messages. If you want to view the logs for a specific service, you can add the service name to the end of the logs command (for example, `docker-compose logs -f app`).

Note: When the app is starting up, it actually sits and waits for MySQL to bbe up and ready before trying to connect to it. Docker doesn't have any built-in support to wait for another container to be fully up, running, and ready before starting another container. For Node-based projects, you can use the `wait-port` dependency. Similar projects exist for other languages/frameworks.

4. At this point, you should be able to open your app and see it running. And hey! We're down to a single command!

#### See the app stack in Docker Dashboard
If we look at the Docker Dashboard, we'll see that there is a group named `app`. This is the "project name" from Docker Compose and used to group the containers together. By default, the project name is simply the name of the directory that the `docker-compose.yml` was located in.

If you twirl down the app, you will see the two containers we defined in the compose file. The names are also a little more descriptive, as they follow the pattern of `<project-name>_<service-name>_<replica-number>`. So it's very easy to quickly see what container is our app and which container is the mysql database.

#### Tear it all down
When you're ready to tear it all down, simply run `docker-compose down` or hit the trash can on the Docker Dashboard for the entire app. The contaienrs will stop and the network will be removed.

**Warning**: By default, named volumes in your compose file are **NOT** removed when running `docker-compose down`. If you want to remove the volumes, you will need to add the `--volumes` flag. The Docker Dashboard does *not* remove volumes when you delete the app stack.

Once torn down, you can switch to another project, run `docker-compose up` and be ready to contribute to that project! It really doesn't get much simpler than that!

### <ins>Image-building best practices
#### Security scanning
When you've built an image, it's good practice to scan it for security vulnerabilities using the `docker scan` command. Docker partnered with `Snyk` to provide vulnerability scanning services. Try scanning `getting-started-docker` image created earlier:
```
docker scan getting-started-docker
```
The scan uses a constantly updated database of vulnerabilities, so the output you see will vary as new vulnerabilities are discovered. An example of potential vulnerabilites you may see:
```
✗ Low severity vulnerability found in freetype/freetype
  Description: CVE-2020-15999
  Info: https://snyk.io/vuln/SNYK-ALPINE310-FREETYPE-1019641
  Introduced through: freetype/freetype@2.10.0-r0, gd/libgd@2.2.5-r2
  From: freetype/freetype@2.10.0-r0
  From: gd/libgd@2.2.5-r2 > freetype/freetype@2.10.0-r0
  Fixed in: 2.10.0-r1

✗ Medium severity vulnerability found in libxml2/libxml2
  Description: Out-of-bounds Read
  Info: https://snyk.io/vuln/SNYK-ALPINE310-LIBXML2-674791
  Introduced through: libxml2/libxml2@2.9.9-r3, libxslt/libxslt@1.1.33-r3, nginx-module-xslt/nginx-module-xslt@1.17.9-r1
  From: libxml2/libxml2@2.9.9-r3
  From: libxslt/libxslt@1.1.33-r3 > libxml2/libxml2@2.9.9-r3
  From: nginx-module-xslt/nginx-module-xslt@1.17.9-r1 > libxml2/libxml2@2.9.9-r3
  Fixed in: 2.9.9-r4
```
The output lists the type of vulnerability, a URL to learn more, and importantly which version of the relevant library fixes the vulnerability.

There are several other options, which you can read about in the [docker scan documentation](https://docs.docker.com/engine/scan/)

As well as scanning your newly built image on the command line, you can also [configure Docker Hub](https://docs.docker.com/docker-hub/vulnerability-scanning/) to scan all newly pushed images automatically, and you can then see the results in both Docker Hub and Docker Desktop.

#### Image layering
Using `docker image history` command, you can see the command that was used to create each layer within an image.
1. Use the `docker image history` command to see the layers in the `getting-started-docker` image you created earlier in the tutorial.
```
docker image history getting-started-docker
```
You should get output that looks something like this (dates/IDs may be different):
```
IMAGE          CREATED        CREATED BY                                      SIZE      COMMENT
88d38af8721f   28 hours ago   /bin/sh -c #(nop)  CMD ["node" "src/index.js…   0B
20e4dce56998   28 hours ago   /bin/sh -c yarn install --production            83.2MB
0447de1d2574   28 hours ago   /bin/sh -c #(nop) COPY dir:00f42896550e119b6…   58.6MB
e2bcfc364290   5 days ago     /bin/sh -c #(nop) WORKDIR /app                  0B
5c6db76c80d7   12 days ago    /bin/sh -c #(nop)  CMD ["node"]                 0B
<missing>      12 days ago    /bin/sh -c #(nop)  ENTRYPOINT ["docker-entry…   0B
<missing>      12 days ago    /bin/sh -c #(nop) COPY file:238737301d473041…   116B
<missing>      12 days ago    /bin/sh -c apk add --no-cache --virtual .bui…   7.62MB
<missing>      12 days ago    /bin/sh -c #(nop)  ENV YARN_VERSION=1.22.5      0B
<missing>      12 days ago    /bin/sh -c addgroup -g 1000 node     && addu…   75.7MB
<missing>      12 days ago    /bin/sh -c #(nop)  ENV NODE_VERSION=12.21.0     0B
<missing>      12 days ago    /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B
<missing>      12 days ago    /bin/sh -c #(nop) ADD file:7eeea546ecde7a036…   5.61MB
```
Each of the lines represents a layer in the image. The display here shows the base at the bottom with the newest layer at the top. Using this, you can also quickly see the size of each layer, helping diagnose large images.

2. You'll notice that several of the lines are truncated. If you add the `--no-trunc` flag, you'll get the full output.
```
docker image history --no-trunc getting-started-docker
```
#### Layer caching
Now that you've seen the layering in action, there's an important lesson to learn to help decrease build times for your container images. **Once a layer changes, all downstream layers have to be recreated as well.**

Let's look at the Dockerfile we were using one more time:
```
FROM node:12-alpine
WORKDIR /app
COPY . .
RUN yarn install --production
CMD ["node", "src/index.js"]
```
Going back to the image history output, we see that each command in the Dockerfile becomes a new layer in the image. You might remember that when we made a change to the image, the yarn dependencies had to be reinstalled. Is there a way to fix this? It doesn't make much sense to ship around the same dependencies every time we build, right?

To fix this, we need to restructure our Dockerfile to help support the caching of the dependencies. For Node-based applications, those dependencies are defined in the `package.json` file. So, what if we copied only that file in first, install the dependencies, and *then* copy in everything else? Then, we only recreate the yarn dependencies if there was a change to the package.json.

1. Update the Dockerfile to copy in the `package.json` first, install dependencies, and then copy everything else in.
```
 FROM node:12-alpine
 WORKDIR /app
 COPY package.json yarn.lock ./
 RUN yarn install --production
 COPY . .
 CMD ["node", "src/index.js"]
```
2. Create a file named `.dockerignore` in the same folder as the Dockerfile with the following contents.
```
node_modules
```
`.dockerignore` files are an easy way to selectively copy only image relevant files. You can read more about this [here](https://docs.docker.com/engine/reference/builder/#dockerignore-file). In this case, the `node_modules` folder should be omitted in the second `COPY` step because otherwise, it would possibly overwrite files which were created by the command in the `RUN` step. For further details on why this is recommended for Node.js applications and other best practices, have a look at their guide on [Dockerizing a Node.js web app](https://nodejs.org/en/docs/guides/nodejs-docker-webapp/).

3. Build a new image using `docker build`.
```
docker build -t getting-started-docker .
```
You should see output like this:
```
Sending build context to Docker daemon  4.661MB
Step 1/6 : FROM node:12-alpine
 ---> 5c6db76c80d7
Step 2/6 : WORKDIR /app
 ---> Using cache
 ---> e2bcfc364290
Step 3/6 : COPY package.json yarn.lock ./
 ---> fff01845f82e
Step 4/6 : RUN yarn install --production
 ---> Running in e25fbcb8a990
yarn install v1.22.5
[1/4] Resolving packages...
[2/4] Fetching packages...
info fsevents@1.2.9: The platform "linux" is incompatible with this module.
info "fsevents@1.2.9" is an optional dependency and failed compatibility check. Excluding it from installation.
[3/4] Linking dependencies...
[4/4] Building fresh packages...
Done in 14.91s.
Removing intermediate container e25fbcb8a990
 ---> 6f2366bb9ece
Step 5/6 : COPY . .
 ---> 84128784bf27
Step 6/6 : CMD ["node", "src/index.js"]c
 ---> Running in aa80605eff79
Removing intermediate container aa80605eff79
 ---> 126dbbd0703c
Successfully built 126dbbd0703c
Successfully tagged getting-started-docker:latest
```
You'll see that all layers were rebuilt. Perfectly fine since we canged the Dockerfile quite a bit.

4. Now, make a change to the `src/static/index.html` file (like change the `<title>` to say "The Awesome Todo App").

5. Build the Docker iamge now using `docker build -t getting-started-docker .` again. This time you output should look a little different.
```
Sending build context to Docker daemon  4.661MB
Step 1/6 : FROM node:12-alpine
 ---> 5c6db76c80d7
Step 2/6 : WORKDIR /app
 ---> Using cache
 ---> e2bcfc364290
Step 3/6 : COPY package.json yarn.lock ./
 ---> Using cache
 ---> fff01845f82e
Step 4/6 : RUN yarn install --production
 ---> Using cache
 ---> 6f2366bb9ece
Step 5/6 : COPY . .
 ---> 2ffeb8b4116c
Step 6/6 : CMD ["node", "src/index.js"]c
 ---> Running in 7fbf8a49a386
Removing intermediate container 7fbf8a49a386
 ---> 4dd27e105b4d
Successfully built 4dd27e105b4d
Successfully tagged getting-started-docker:latest
```
First off, you should notice that the build was MUCH faster! And you'll see that steps 1-4 all have `Using cache`. So hooray! We're using the build cache. Pushing and pulling this image and updates to it will be much faster as well. Yay!

#### Multi-stage builds (examples below)
While we're not going to dive into it too much in this tutorial, multi-stage builds are an incredibly powerful tool to help use multiple stages to create an image. There are several advantages for them:
* Separate build-time dependencies from runtime dependencies
* Reduce overall image size by shipping *only* what your app needs to run

1. Maven/Tomcat example

When building Java-based applications, a JDK is needed to compile the source code to Java bytecode. However, that JDK isn't needed in production. Also, you might be using tools like Maven or Gradle to help build the app. Those also aren't needed in our final image. Multi-stage builds help.
```
FROM maven AS build
WORKDIR /app
COPY . .
RUN mvn package

FROM tomcat
COPY --from=build /app/target/file.war /usr/local/tomcat/webapps
``` 
In this example, we use one stage (called `build`) to perform the actual Java build using Maven. In the second stage (starting at `FROM tomcat`), we copy in files from the `build` stage. The final image is only the last stage being created (which can be overridden using the `--target` flag).

2. React example

When building React applications, we need a Node environment to compile the JS code (typically JSX), SASS stylesheets, and more into static HTML, JS, and CSS. If we aren't doing server-side rendering, we don't even need a Node environment for our production build. Why not ship the static resources in a static nginx container?
```
FROM node:12 AS build
WORKDIR /app
COPY package* yarn.lock ./
RUN yarn install
COPY public ./public
COPY src ./src
RUN yarn run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
```
Here, we are using a `node:12` image to perform the build (maximizing layer caching) and then copying the output into an nginx container. Pretty neat!

### What next?
Although we're done with this workshop, there's still a LOT mroe to learn about containers! A few other areas to look at next are:

#### Container orchestration
Running containers in production is tough. You don’t want to log into a machine and simply run a `docker run` or `docker-compose up`. Why not? Well, what happens if the containers die? How do you scale across several machines? Container orchestration solves this problem. Tools like Kubernetes, Swarm, Nomad, and ECS all help solve this problem, all in slightly different ways.

The general idea is that you have “managers” who receive **expected state**. This state might be “I want to run two instances of my web app and expose port 80.” The managers then look at all of the machines in the cluster and delegate work to “worker” nodes. The managers watch for changes (such as a container quitting) and then work to make **actual state** reflect the expected state.

#### Cloud Native Computing Foundation projects
The CNCF is a vendor-neutral home for various open-source projects, including Kubernetes, Prometheus, Envoy, Linkerd, NATS, and more! You can view the [graduated and incubated projects here](https://www.cncf.io/projects/) and the entire [CNCF Landscape here](https://landscape.cncf.io/). There are a LOT of projects to help solve problems around monitoring, logging, security, image registries, messaging, and more!


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