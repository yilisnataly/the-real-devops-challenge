# Solutions in relation to the real DevOps challenge

# Challenge 3. Dockerize the APP

A docker image is built through Dockerfile in order to run a docker container for flask app.
Note that we have used alpine image for light weight

```bash
FROM python:3.9-alpine

WORKDIR /app

COPY ./requirements.txt /app/requirements.txt

RUN pip install --no-cache-dir -r /app/requirements.txt

COPY ./ /app

ENV FLASK_APP=app.py
ENV FLASK_ENV=development
ENV FLASK_RUN_HOST=0.0.0.0

CMD ["python3", "app.py"]
```

We have run the docker build command to create the app image.
```bash
docker build -t ramirezy/flask-challenge:1.0 .
```
As next step, we upload to Docker Hub
```bash
docker push ramirezy/flask-challenge:1.0
```
Checking the image has been successfully created
```bash
docker images
REPOSITORY                      TAG          IMAGE ID       CREATED        SIZE
ramirezy/flask-challenge        1.0          f606b96795a3   3 hours ago    191MB
```

# Challenge 4. Dockerize the database
Once the app image is built, we will need to launch a mongodb container with the database connection parameters with the following command:
```bash
sudo docker run -d --name mongodb -p 27017:27017 -v mongo_db:/data/db \
  -e MONGO_INITDB_DATABASE=restaurantdb \
  -e MONGO_INITDB_ROOT_USERNAME=user_root \
  -e MONGO_INITDB_ROOT_PASSWORD=root_password \
     mongo
 ```
# Challenge 5. Docker Compose it
The docker compose is created to deploy the application and the database with the corresponding environment variables.

In the `docker-compose.yml` file, the values of the environment variables have been parameterized with a file called file.env, but it has been included in the `.gitignore` file, since it is vulnerable information of the database.
```bash
version: '3.9'

services:
  app:
    build: .
    command: python -u app.py
    #image: ramirezy/flask-challenge
    container_name: flask
    restart: unless-stopped
    environment:
     MONGO_INITDB_ROOT_USERNAME: ${MONGO_INITDB_ROOT_USERNAME}
     MONGO_INITDB_ROOT_PASSWORD: ${MONGO_INITDB_ROOT_PASSWORD}
     MONGO_INITDB_DATABASE: ${MONGO_INITDB_DATABASE}
     MONGODB_HOSTNAME: ${MONGODB_HOSTNAME}
     MONGODB_PORT: ${MONGODB_PORT}
    ports:
     - 8080:8080
    depends_on:
      - mongodb
    volumes:
      - ./app.py:/app/app.py
  mongodb:
    image: mongo:5.0.9
    container_name: mongodb
    restart: unless-stopped
    command: mongod --auth
    environment:
     MONGO_INITDB_ROOT_USERNAME: ${MONGO_INITDB_ROOT_USERNAME}
     MONGO_INITDB_ROOT_PASSWORD: ${MONGO_INITDB_ROOT_PASSWORD}
     MONGO_INITDB_DATABASE: ${MONGO_INITDB_DATABASE}

    ports:
     - 27017:27017
    volumes:
     - /mongo_db:/data/db

volumes:
 mongo_db:
 ```
To run it, we type the command `docker-compose --env-file file.env up -d`

Now we can test the endpoint by typing the URL `http://127.0.0.1:8080/api/v1/restaurant` on your browser, but you will not see anything due we will need to insert the database data. 

Once the build process is completed, we launch the following command to list the running containers:
```bash
docker ps                                                                                                                          
CONTAINER ID   IMAGE                           COMMAND                  CREATED       STATUS       PORTS                                           NAMES
348cb7a987f2   the-real-devops-challenge_app   "python -u app.py"       4 hours ago   Up 4 hours   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp       flask
7dd46da27695   mongo:5.0.9                     "docker-entrypoint.s…"   4 hours ago   Up 4 hours   0.0.0.0:27017->27017/tcp, :::27017->27017/tcp   mongodb
```
We access the mongo container to create the user and any other parameters for the database.
```bash
docker exec -it mongodb bash
```
Once inside we log in the Mongodb shell with authentication
```bash
oot@7dd46da27695:/# mongo -u user_root -p root_password --authenticationDatabase admin restaurantdb
MongoDB shell version v5.0.9
connecting to: mongodb://127.0.0.1:27017/restaurantdb?authSource=admin&compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("3484164e-9da1-4719-864a-f7f2fd6430e5") }
MongoDB server version: 5.0.9
================
Warning: the "mongo" shell has been superseded by "mongosh",
which delivers improved usability and compatibility.The "mongo" shell has been deprecated and will be removed in
an upcoming release.
For installation instructions, see
https://docs.mongodb.com/mongodb-shell/install/
================
```
We double checked the database has been created
```bash
show dbs
admin         0.000GB
config        0.000GB
local         0.000GB
restaurantdb  0.000GB
```
We created the user as follows:
```bash
db.createUser({
  user: 'user_root',
  pwd: 'root_password',
  roles: [
    {
      role: 'readWrite',
      db: 'restaurantdb',
    },
  ],
});
```
The collection is created with the name `restaurant` as requested `db.createCollection("restaurant")`

As next step we will import the restaurant dataset to load the mongodb collection, hence we copy it into the mongo container.
```bash
docker cp restaurant.json mongodb:/
```
We import the collection with the following command shown below:
```bash
mongoimport --username user_root --password root_password --authenticationDatabase admin --db restaurantdb --collection restaurant --file restaurant.json
```
