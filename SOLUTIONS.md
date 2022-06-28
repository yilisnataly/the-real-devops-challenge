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
Once the app image is built, we will need to launch a container with the database connection parameters with the following command:
```bash
sudo docker run -d --name mongodb -p 27017:27017 -v mongo_db:/data/db \
  -e MONGO_INITDB_DATABASE=db_name \
  -e MONGO_INITDB_ROOT_USERNAME=user_root_password \
  -e MONGO_INITDB_ROOT_PASSWORD=root_password \
     mongo
 ```
