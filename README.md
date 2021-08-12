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
   - To create the image (overriding the default port just to test)
      - `docker run -p 3000:8081 --env-file ./.env -d --name test-api --rm -v $(pwd):/app:ro docker-test-api:initial`
