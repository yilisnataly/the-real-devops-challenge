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
# Running the app
After we have inserted all the required data in the database container, we can proceed to run it.
```bash
http://127.0.0.1:8080/api/v1/restaurant
```
![endpoint_api](https://user-images.githubusercontent.com/39458920/176285697-52e4a4f9-18c6-4280-9a4e-71cc72a50a83.JPG)

# Final Challenge. Deploy it on kubernetes
Checking that all manifests have been deployed in Kubernetes and they are running properly.

```bash
ubectl get all -n flask-app             
NAME                                       READY   STATUS    RESTARTS   AGE
pod/flaskapp-deployment-5c4b7bb67f-h7tfc   1/1     Running   0          4h17m
pod/mongodb-76c4459dd7-vvzpp               1/1     Running   0          9h

NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/flask-service   LoadBalancer   10.245.55.147   167.99.17.1   8080:32203/TCP   9h
service/mongodb         ClusterIP      None            <none>        27017/TCP        9h

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/flaskapp-deployment   1/1     1            1           4h28m
deployment.apps/mongodb               1/1     1            1           9h

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/flaskapp-deployment-5c4b7bb67f   1         1         1       4h17m
replicaset.apps/flaskapp-deployment-7c96fc7b8    0         0         0       4h28m
replicaset.apps/mongodb-76c4459dd7               1         1         1       9h

NAME                                           REFERENCE                        TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/flask-ha   Deployment/flaskapp-deployment   <unknown>/80%   1         10        1          3h55m
```
<b>Ingress Controller installation</b>

The ingress controller will be used to be able to access to the application, which it works as inverse proxy, so will have to install it in our kubernetes cluster.

The ingress controller can be found in the offical web <a href="https://kubernetes.github.io/ingress-nginx/deploy/#digital-ocean">here </a> </li>, as per the cluster provider. In our case it's should be specified for Digital Ocean:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.0/deploy/static/provider/do/deploy.yaml
```

As result, a namespace is created as `ingress-nginx` where we can find all resources created.

<b>Host service Host nip.io</b>

Now we need to deploy the ingress object to relate the host where we will be accessing toward our app, so the DNS resolution service nip.io will be used.

In the manifest below we reference the host as `practica.64.225.82.140.nip.io` which will resolve the mentioned IP with the service nip.io. For further info you can check <a href="https://nip.io/">here </a> </li>

Beside, it is also needed to indicate the annotation: `nginx.ingress.kubernetes.io/rewrite-target: /api/v1/restaurant`, since the app response to the endpoint `/api/v1/restaurant`

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: flask-ingress
  namespace: flask-app
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /api/v1/restaurant
spec:
  rules:
  - host: practica.64.225.82.140.nip.io
    http:
      paths:
      - path: '/'
        pathType: Prefix
        backend:
          service:
            name: flask-service
            port:
              number: 8080
 ```
