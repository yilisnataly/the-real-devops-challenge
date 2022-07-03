# Solutions in relation to the real DevOps challenge

# Challenge 2. Test the application in any cicd system
The application has been tested with GitHub actions, s
```bash
flake8:
    name: Code Quality
    runs-on: ubuntu-latest
    steps:
    - name:  Checkout code
      uses: actions/checkout@v2
    
    - name: Set up Python 3.9
      uses: actions/setup-python@v2
      with:
        python-version: 3.9
    - name: Lint with flake8
      run: |
        pip install flake8 flake8-html
        # stop the build if there are Python syntax errors or undefined names
        flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
        # exit-zero treats all errors as warnings. The GitHub editor is 127 chars wide
        mkdir -p reports/flake8
        flake8 . --count --exit-zero --max-complexity=10 --max-line-length=79 --statistics --format=html --htmldir=reports/flake8
    - name: Archive flake8 coverage results
      uses: actions/upload-artifact@v2
      with:
        name: flake8-coverage-report
        path: reports/flake8/
  ```
  
  ```bash
   pytest:
    name: Unit Testing
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Set up Python
      uses: actions/setup-python@v2
      with:
        python-version: 3.9
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements_dev.txt
    - name: Test with pytest
      run: |
        pip install pytest pytest-cov pytest-html pytest-sugar pytest-json-report
        pytest -v --cov --html=reports/pytest/report.html
    - name: Archive pytest coverage results
      uses: actions/upload-artifact@v2
      with:
        name: pytest-coverage-report
        path: reports/pytest
     ```

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

We have to run the docker build command to create the app image.
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

Now we can test the endpoint by typing the URL `http://127.0.0.1:8080/api/v1/restaurant` on the browser, but we will not see anything due we will need to insert the database data. 

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

<b>Prerequisite:</b> A Kubernetes cluster needs to be created, hence we have enabled it through a managed Kubernetes service by <b>Digital Ocean</b>. Here, is being used a cluster consisting 3 workers.
```bash
kubectl get nodes
NAME                   STATUS   ROLES    AGE     VERSION
pool-fgl2z6ddg-ckj2b   Ready    <none>   2d13h   v1.22.8
pool-fgl2z6ddg-ckj2j   Ready    <none>   2d13h   v1.22.8
pool-fgl2z6ddg-ckj2r   Ready    <none>   2d13h   v1.22.8
```
To deploy our application on a Kubernetes cluster, we will be creating <b>.yaml</b> files for each Kubernetes resource. The resources are to be deployed as follows:

To run the deployment file we need to create the configmap and the secret to store the vulnerable data. Regarding the secret info should be encoded with Base64.

```bash
apiVersion: v1
data:
  dbname: restaurantdb
  host: mongodb
  port: "27017"
kind: ConfigMap
metadata:
  name: flaskapp-cm
  namespace: flask-app
```
Note that the secret was added in the .gitignore file to not make it visible.

```bash
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret
  namespace: flask-app
type: Opaque
data:
  rootpassword:
  username:
```
# Running Flask app in Kubernetes

Now we set the environment variables in this deployment, using the values specified in the secret and configmap file. The flask image was built from Dockerfile configuration. The spec section defines the pod where we specify the image to be pulled and run. The port 8080 of the Pod is exposed.

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flaskapp-deployment
  namespace: flask-app
  labels:
    app: flaskapp
spec:
  selector:
    matchLabels:
      app: flaskapp
  replicas: 1
  template:
    metadata:
      labels:
        app: flaskapp
    spec:
      containers:
        - name: flask
          image: ramirezy/flask-challenge:1.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: username
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: rootpassword
            - name: MONGO_INITDB_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: flaskapp-cm
                  key: dbname
            - name: MONGODB_HOSTNAME
              valueFrom:
                configMapKeyRef:
                  name: flaskapp-cm
                  key: host
            - name: MONGODB_PORT
              valueFrom:
                configMapKeyRef:
                  name: flaskapp-cm
                  key: port
  ```
The LoadBalancer Service enables the pods in a deployment to be accessible from outside the cluster. The advantage of using a Service is that it gives us a single consistent IP to access our app as many pods may come and go in our deployment.
  
 ```bash
 apiVersion: v1
kind: Service
metadata:
  name: flask-service
  namespace: flask-app
  labels:
    app: flaskapp
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: flaskapp
  type: LoadBalancer
  ```
 # Running Mongodb in Kubernetes
 Now let's provide persistence to the database with Persistent Volume Claim. For this is needed to check the storage classes from the cluster.
 According to the output below, it seems to be `do-block-storage`:
 
```bash
kubectl get storageclasses.storage.k8s.io
NAME                         PROVISIONER                 RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
do-block-storage (default)   dobs.csi.digitalocean.com   Delete          Immediate           true                   2d13h
```

Now we can define the manifest for Persistent Volume Claim:

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pv-claim
  namespace: flask-app
spec:
  storageClassName: do-block-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
 ```
 As you can see Persistent Volume Claim has been successfully created and bound.

```bash
kubectl get pvc -n flask-app
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
mongo-pv-claim   Bound    pvc-a156d1cb-a9a5-430f-a841-9d0461f03a67   5Gi        RWO            do-block-storage   45h
```

Additionally, we can see persistent volume is automatically created.
```bash
kubectl get persistentvolume
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                      STORAGECLASS       REASON   AGE
pvc-a156d1cb-a9a5-430f-a841-9d0461f03a67   5Gi        RWO            Delete           Bound    flask-app/mongo-pv-claim   do-block-storage            45h
```
The `deployment-mongo.yaml` is where we define the mongo deployment that creates a single instance of MongoDB server. Here, we expose the port 27017 which can be accessed by other pods. We also defined the database connection parameters through environment variables.

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
  namespace: flask-app
spec:
  selector:
    matchLabels:
      app: mongodb
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - image: mongo:5.0.9
        name: mongo
        env:
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: rootpassword
        - name: MONGO_INITDB_DATABASE
          valueFrom:
            configMapKeyRef:
              name: flaskapp-cm
              key: dbname
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: username
        ports:
        - containerPort: 27017
          name: mongo
        volumeMounts:
        - name: mongo-persistent
          mountPath: /data/db
      volumes:
      - name: mongo-persistent
        persistentVolumeClaim:
          claimName: mongo-pv-claim
 ```
 Creating a service to provide MongoDB access towards Flask app or any other pod inside the cluster.

 ```bash
 apiVersion: v1
kind: Service
metadata:
  name: mongodb
  namespace: flask-app
spec:
  ports:
  - port: 27017
  selector:
    app: mongodb
  clusterIP: None
  ```

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
# Inserting data in Kubernetes POD
We check the Mongo POD and proceed to access it to create the database `restaurantdb`, create the user, and import the dataset called `restaurant`

```bash
kubectl -n flask-app exec --stdin --tty mongodb-76c4459dd7-vvzpp -- /bin/sh
```

# Ingress Controller installation

The ingress controller will be used to be able to access to the application, which it works as inverse proxy, so will have to install it in our kubernetes cluster.

The ingress controller can be found in the offical web <a href="https://kubernetes.github.io/ingress-nginx/deploy/#digital-ocean">here </a> </li>, as per the cluster provider. In our case it's should be specified for Digital Ocean:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.0/deploy/static/provider/do/deploy.yaml
```

As result, a namespace is created as `ingress-nginx` where we can find all resources created.

# Host service Host nip.io

Now we need to deploy the ingress object to relate the host where we will be accessing toward our app, so the DNS resolution service <b>nip.io</b> will be used.

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

Checking whole the ingress resources

```bash
kubectl get all --namespace=ingress-nginx
NAME                                            READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create--1-g8c7b     0/1     Completed   0          41h
pod/ingress-nginx-admission-patch--1-6jdkg      0/1     Completed   0          41h
pod/ingress-nginx-controller-5849c9f946-w926r   1/1     Running     0          41h

NAME                                         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.245.2.243    64.225.82.140   80:31422/TCP,443:31281/TCP   41h
service/ingress-nginx-controller-admission   ClusterIP      10.245.50.161   <none>          443/TCP                      41h

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           41h

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-5849c9f946   1         1         1       41h

NAME                                       COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   1/1           3s         41h
job.batch/ingress-nginx-admission-patch    1/1           4s         41h
```
We check the host and verified that we are able to access it.
```bash
kubectl get ingress -n flask-app  
NAME            CLASS    HOSTS                           ADDRESS         PORTS   AGE
flask-ingress   <none>   practica.64.225.82.140.nip.io   64.225.82.140   80      41h
```
![endpoint_k8s](https://user-images.githubusercontent.com/39458920/176661829-9648703c-90c7-47ac-afae-85fd3737793f.JPG)

# Applying manifests
```bash
kubectl apply -f ./k8s  
```
# Removing resources
```bash
kubectl delete -f ./k8s
```
