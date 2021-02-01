FROM node:14
RUN mkdir -p /app && chown -R node:node /app
WORKDIR /app
USER node

RUN mkdir tmp
RUN mkdir assets

COPY ./package*.json /app/
RUN npm install

COPY --chown=node:node *.js /app/
COPY --chown=node:node assets/* /app/assets/
COPY --chown=node:node tmp /app/

CMD ["node", "/app"]

## docker network create k8scourse

## cd db
## RUN MYSQL DATABASE with NETWOKR
## docker run -p 3306:3306 -d --name k8scourse-db -e MYSQL_USER=user -e MYSQL_PASSWORD=mysql -e MYSQL_ROOT_PASSWORD=root --rm -v $(pwd)/init.sql:/docker-entrypoint-initdb.d/init.sql --network k8scourse mysql:5.7

## RUN Backend with NETWORK
## docker run  -p 3000:3000 -d --name back --rm -e MYSQL_HOST=k8scourse-db --network k8scourse k8scourse-back

## docker build -t k8scourse-back .
## docker run -p 3000:3000 --name back -d --rm k8scourse-back
