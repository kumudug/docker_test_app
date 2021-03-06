# App to test Docker

* This app contains a
   - Node.js backend
   - React frontend
   - MongoDB database

## Create a network
   - We will be running 3 containers. One for each component mentioned above. Thus need a docker network with the bridge driver
      - `docker network create test-network`

## Create the MongoDB container
   - Now create a MonogoDB container using an official image in the created network
      - `docker run -d --name mongoctr --network test-network mongo:5.0.2`
      - See below for adding credentials and bind mounts to mongo container

## Create the backend container.

* Created `.env` file to use as a store for environment variables
   - This way I don't have to use `-e PORT=8080 -e ANOTER_VAR=30`
   - Instead I can define all environment variables and use it as `--env-file ./.env`
* Specified an `ARG` which can be used to create different images for dev and prod
   ```
   # Set build time argument
   ARG DEFAULT_PORT=80
   ```
   - This can be used as
      - `docker build . -t testapp-backend:dev --build-arg DEFAULT_PORT=8080`
      - `docker build . -t testapp-backend:prod --build-arg DEFAULT_PORT=80`

* Used a environment variable based on the arg
   - This means the port can be changed after image creation during container creation.
   - This kind of makes the ARG redundant but did it just for testing.
   ```
   # Set environment variable based on ARG to use in nodejs
   ENV PORT $DEFAULT_PORT
   ```
* Added an anonymous volume for the NODE_MODULES folder
   ```
   # Anonymous volume for the node_modules folder
   VOLUME [ "/app/node_modules" ]
   ```
* Going to use a bind mount to auto refresh code with nodemon
   - Build the container
      - `docker build . -t docker-test-api:initial --build-arg DEFAULT_PORT=8080`
   - To create the image (overriding the default port just to test), use a named volume for the logs folder
      - `docker run -p 3001:8081 --network test-network --env-file ./.env -d --name test-api --rm -v api-logs:/app/logs -v $(pwd):/app:ro docker-test-api:initial`
      - Code refresh caused a permission issue. Removed the bind mount and make docker run `node app.js` instead. Also removed the `--rm` flag
      - `docker run -p 3001:8081 --network test-network --env-file ./.env -d --name test-api -v api-logs:/app/logs docker-test-api:initial`
   - To create the image without exposing the api for the local machine
      - This way the API is only accessible within the docker network
      - `docker run --network test-network --env-file ./.env -d --name test-api -v api-logs:/app/logs docker-test-api:initial`

### Getting the realtime code refresh working for API project

* Last time code refresh cause a permission issue thus removed the bind mount for code and now running `node app.js` in the Docker file.
* Lets add the code as a bind mount again and change the code and test.
   - `docker run --network test-network --env-file ./.env -d --name test-api -v $(pwd):/app:ro -v api-logs:/app/logs docker-test-api:initial`
   - This kept giving `permission denied, open '/app/logs/access.log'`
   - Got it working without the named volume and the physical logs folder permission set to write all `sudo chmod ugo+rwx ./app/logs` and remove `:ro` from bind mount
   - `docker run --network test-network --env-file ./.env -d --name test-api -v $(pwd):/app docker-test-api:initial`

### Start setting db username and pwd in command prompt

* This ideally should be done using docker secrets. 
    - `docker run --network test-network --env-file ./.env -e DBUSER=mongoadmin -e DBPWD=secret -d --name test-api -v $(pwd):/app docker-test-api:initial`

## Create the frontend container

* Created a Dockerfile
* Created an ARG, ENV variable for api port and replaced that in the App.js
* Build the container
   - `docker build . -t docker-test-app:initial --build-arg DEFAULT_PORT=3000 --build-arg DEFAULT_API_PORT=3001`
* Create and run the image
   - We need to add `-it` interactive flag in order to keep the react dev server running
   - `docker run -p 8080:3000 --network test-network --env-file ./.env -d --name test-app -it --rm docker-test-app:initial`
* In order to get node environment variables to work with the react scripts
   - Remove `.env` file from the `.dockerignore`
   - Duplicate the environment files with `REACT_APP_`. Ex: `REACT_APP_APIPORT=3001`
   - This makes the react-scripts read the .env file and load them in
* To access the DB from within the network
   - Change the `localhost` to `test-api` as it's the container name
   - Change the port to 8081 as it's the port exposed to the docker network
   - Build and run again
   `docker build . -t docker-test-app:initial --build-arg DEFAULT_PORT=3000 --build-arg DEFAULT_API_PORT=8081`
   - `docker run -p 8080:3000 --network test-network --env-file ./.env -d --name test-app -it --rm docker-test-app:initial`
   - The DB url need to be fixed
      - Remember react runs in the browser not in the container. So the container name in the url is not understood by react.
      - The only thing running in the container is the development server
      - For the time being we are passing the api docker ip in a hacky way via an env variable
      - You can find the api containers ip using the `docker container inspect` command
      - Add the environment variable to the `env` file.
      - `REACT_APP_API_IP=172.19.0.3`
      - `docker run -p 8080:3000 --network test-network --env-file ./.env -d -e REACT_APP_APIIP=172.19.0.3 --name test-app -it --rm  docker-test-app:initial`

### Source code refresh for frontend app

* Add a bind mount to the source folder
   - `docker run -p 8080:3000 --network test-network -v $(pwd)/src:/app/src:ro --env-file ./.env -d -e REACT_APP_APIIP=172.19.0.3 --name test-app -it --rm  docker-test-app:initial`
   - We don't need nodemon as `react-scripts` handles file watch

## Making the MongoDB data persist outside of container life cycles

### Using a named volume

* According to MongoDB official [image](https://hub.docker.com/_/mongo) we can map a volume to /data/db to override where the data is stored.
    - Here we are using a named docker volume to store the data. This way we can recreate the mongo container used for the app without loosing data
* Steps
   - Stop and delete the running mongo container 
      -`docker stop mongoctr`
      - `docker rm mongoctr`
   - Start with a named volume `docker run -d --name mongoctr -v data:/data/db --network test-network mongo:5.0.2`
   - Test by going to `http://localhost:8080/`

### Using a bind mount
   * We can also use a bind mount. This way the data directory can be easily accessed by tools
         -`docker stop mongoctr`
         - `docker rm mongoctr`
         - `docker run -d --name mongoctr -v $(pwd)/data:/data/db --network test-network mongo:5.0.2`

## Securing the MongoDB db with credentials

* We can specify a user name and a password for the db [image docs](https://hub.docker.com/_/mongo)
   - `docker stop mongoctr`
   - `docker rm mongoctr`
   - `docker run -d --name mongoctr -v $(pwd)/data:/data/db --network test-network -e MONGO_INITDB_ROOT_USERNAME=mongoadmin -e MONGO_INITDB_ROOT_PASSWORD=secret mongo:5.0.2`
   - NOTE: Use docker secrets instead of passing the user and pwd in command line
* We need to change the connection string used in the api app and rebuild it now
   - `mongodb://mongoadmin:secret@mongoctr:27017/course-goals?authSource=admin`

## Using Docker Compose

* Added a `docker-compose.yml`
   - Added config for dynamo container
* Start
   - `docker-compose up -d`
* Stop 
   - `docker-compose down`
