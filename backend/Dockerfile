FROM node:16

WORKDIR /app

COPY package.json .

RUN npm install

COPY . .

# Set build time argument
ARG DEFAULT_PORT=80

# Set environment variable based on ARG to use in nodejs
ENV PORT $DEFAULT_PORT

EXPOSE $PORT

# Anonymous volume for the node_modules folder
VOLUME [ "/app/node_modules" ]

CMD ["npm", "start"]