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
      - `docker run -p 3001:8081 --network test-network --env-file ./.env -d --name test-api -v api-logs:/app/log docker-test-api:initial`

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
* The URL of the front end app currently uses the `localhost` not the actual defined Docker network. When I tried using the api app name `test-api` instead of `localhost` I got intermittent failures.