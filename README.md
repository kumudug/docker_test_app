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