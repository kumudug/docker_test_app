FROM node:16

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm install

COPY . .

# Set build time argument
ARG DEFAULT_PORT=3000
ARG DEFAULT_API_PORT=3001

# Set environment variable based on ARG to use in nodejs
ENV PORT $DEFAULT_PORT
ENV APIPORT $DEFAULT_API_PORT

EXPOSE $PORT

# Anonymous volume for the node_modules folder
VOLUME [ "/app/node_modules" ]

CMD ["npm", "start"]