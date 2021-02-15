#  Build Docker image for your frontend application

The Desotech 3-tiers will be:

- Frontend tier: This will host the web application.
- Middle tier: This will host the api, in our case the REST api.
- Database tier: This will host the database.

![](custom_themes/assets/images/diagramma.png)

```
cd ~
git clone https://github.com/desotech-it/desotech-3-tier-app.git
cd desotech-3-tier-app
```

## The Application

We are going to build a web application that will display a list of countries and their capitals in a plain old html table. This applications is going to get this data from a rest api which in turn will fetch it from a postgresql database. Its a dead simple app partitioned into 3-tiers. We are going to build the webapp and the rest api in node. Note that this is not going to be a primer on node apps. I assume that the reader is comfortable with creating node webapps.

File: /home/student/desotech-3-tier-app/db/Dockerfile

```
FROM postgres:13-alpine
ENV POSTGRES_USER deso_user
ENV POSTGRES_PASSWORD deso_user
ENV POSTGRES_DB deso_db
COPY init_sql_scripts/init.sql /docker-entrypoint-initdb.d/init.sql
```

```
docker build -t db:1.0 .
```

> ```
> Successfully built 14a9a296199c
> Successfully tagged db:1.0
> ```

```
docker image ls
```

> ```
> REPOSITORY                              TAG                     IMAGE ID            CREATED             SIZE
> db                                      1.0                     14a9a296199c        48 minutes ago      158MB
> postgres                                13-alpine               329922c3c1c1        4 weeks ago         158MB
> ```


For permit docker to resolve correctly the name we can create a docker network for frontend and backend:

```
docker network create frontend
docker network create backend
```

> ```
> 7b92257c7c2110aaa6636a4c935ecd1cbb1822ef049d4ff5314ec23b1e14e660
> a99ead0d640f69701ec8b9fe6b9fa8f7d87e9ef1c8067b485135b8c862351707
> ```

```
docker container run --net backend --name db --hostname db -d db:1.0
```

> ```
> 7bc394d5a84a6d9bc8494594fd4ed6d9239a1a6517110de0155887c7fb24c119
> ```

```
docker container ls
```

> ```
> CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
> 7bc394d5a84a        db:1.0              "docker-entrypoint.s…"   25 minutes ago      Up 25 minutes       5432/tcp                 db
> ```

```
docker container logs db
```

> ```
> /usr/local/bin/docker-entrypoint.sh: running /docker-entrypoint-initdb.d/init.sql
> CREATE TABLE
> INSERT 0 1
> INSERT 0 1
> INSERT 0 1
> INSERT 0 1
> . . .
> 2020-09-16 17:14:52.412 UTC [1] LOG:  database system is ready to accept connections
> ```

## Write Dockerfile for your backend application


```
cd ~/desotech-3-tier-app/api
cat index.js
```

Its a simple app listening on port 3001. We are using express and pg packages. You will have to install them. We are simply fetching data from the country_and_capitals table and sending all the rows as response when we get a GET request on the /data route. The connection string is being picked from the environment variable `CONNECTION_STRING`. It is of the format `postgres://username:password@host/dbname`.

Now that we have our app, we proceed to containerise it. Here goes the Dockerfile for this app.


File: /home/student/desotech-3-tier-app/api/Dockerfile

```
FROM node:10-alpine

WORKDIR /usr/src/app

COPY package*.json ./

RUN npm install
COPY . .

EXPOSE 3001
CMD [ "node", "index.js" ]
```

We are building this in layers. We are basing our image from a docker alpine node image. We are then setting /usr/src/app as the working directory for the next two commands. We issue COPY command to copy package.json and package-lock.json to the working directory and then do a npm install. This will install all the dependencies. Notice that we have still not copied the necessary node code yet, we do this so that docker can cache the intermediate layers. Then we copy the directory contents to the image. We expose the 3001 port and then do a node index.js command using the docker CMD command.

```
docker build -t api:1.0 .
```

> ```
> Successfully built ed543ae412a3
> Successfully tagged api:1.0
> ```

```
docker image ls
```

> ```
> REPOSITORY                              TAG                     IMAGE ID            CREATED             SIZE
> api                                     1.0                     ed543ae412a3        38 minutes ago      86.4MB
> db                                      1.0                     14a9a296199c        48 minutes ago      158MB
> node                                    10-alpine               3ac6f53a4041        3 hours ago         83.5MB
> postgres                                13-alpine               329922c3c1c1        4 weeks ago         158MB
> ```


```
docker create --net backend --name api --hostname api -e CONNECTION_STRING=postgres://deso_user:deso_pass@db/deso_db -p 3001:3001 api:1.0
```

> ```
> daaf9defa47970b44ef212a7bb98a40ca0da22950d9bcd2bca8bd024712ce92e
> ```

```
docker network connect frontend api
docker container start api
```


```
docker container ls
```

> ```
> CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
> 18e04a8053bd        api:1.0             "docker-entrypoint.s…"   20 minutes ago      Up 20 minutes       0.0.0.0:3001->3001/tcp   api
> 7bc394d5a84a        db:1.0              "docker-entrypoint.s…"   25 minutes ago      Up 25 minutes       5432/tcp                 db
> ```

```
docker container logs api
```

> ```
> Backend rest api listening on port 3001!
> ```

Curl the api endpoint http://localhost:3001/data. You should see a json response like this:

```
curl http://localhost:3001/data
```

> ```json
> {"data":[{"country":"Italia","capital":"Roma"},{"country":"France","capital":"Paris"},{"country":"UK","capital":"London"},{"country":"Germany","capital":"Berlin"}]}%
> ```

Last step will be create the frontend tier.

## Frontend tier

```
cd ~/desotech-3-tier-app/frontend
cat index.js
```

This app is calling the REST api we created earlier using request package and displaying the data as a table. The api url is taken from a environment variable. The Dockerfile for this app is exactly the same from the middle tier but for the port. We are using the port 3000.

```
FROM node:10-alpine

WORKDIR /usr/src/app

COPY package*.json ./

RUN npm install
COPY . .

EXPOSE 3000
CMD [ "node", "index.js" ]
```



```
docker build -t frontend:1.0 .
```

> ```
> Successfully built 289d33c6166a
> Successfully tagged frontend:1.0
> ```

```
docker image ls
```

> ```
> REPOSITORY                              TAG                     IMAGE ID            CREATED             SIZE
> frontend                                1.0                     289d33c6166a        12 seconds ago      90.6MB
> api                                     1.0                     ed543ae412a3        38 minutes ago      86.4MB
> db                                      1.0                     14a9a296199c        48 minutes ago      158MB
> node                                    10-alpine               3ac6f53a4041        3 hours ago         83.5MB
> postgres                                13-alpine               329922c3c1c1        4 weeks ago         158MB
> ```

```
docker run -d --net frontend --name frontend --hostname frontend -e API_URL=http://api:3001/data -p 80:3000 -d frontend:1.0
```

> ```
> 65c06ebad2c10ef700380c55530720531b07285536a701d2339c7cc909150f8c
> ```

```
docker container ls
```

> ```
> CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
> 65c06ebad2c1        frontend:1.0        "docker-entrypoint.s…"   10 seconds ago      Up 9 seconds        0.0.0.0:80->3000/tcp     frontend
> 18e04a8053bd        api:1.0             "docker-entrypoint.s…"   20 minutes ago      Up 20 minutes       0.0.0.0:3001->3001/tcp   api
> 7bc394d5a84a        db:1.0              "docker-entrypoint.s…"   25 minutes ago      Up 25 minutes       5432/tcp                 db
> ```

```
curl localhost
```

> ```html
> <table border="1"><tr><td>Country</td><td>Capital</td></tr><tr><td>Italia</td><td>Roma</td></tr><tr><td>France</td><td>Paris</td></tr><tr><td>UK</td><td>London</td></tr><tr><td>Germany</td><td>Berlin</td></tr></table>%
> ```

If your application don't show this output, please advise your instructor.
