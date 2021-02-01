# Lesson 7 - Using Environment Variables In Front-End Applications

# Mini talk intro

In the previous lesson, we managed to put all of our front-end in a container and we can now distribute this container to the rest of our team. This will work well with stand-alone websites but as soon as you need to connect to an API, this could become an issue since the URL that you use in development will most likely differ from the one in production. 

In order for your code to still work in production, you will need to specify the base URL of your server. As you've seen in a previous lesson, this can be done by using an environment variable. Since the value of the base URL changes between the environments that you are in, it would be the best way to actually set this. 

But there is a problem when you want to use environment variables in your front-end. When you used an environment variable for the back end, it works because the Node.js server was able to access this value from the operating system. In the case of your front-end it lives in the browser of your end-users. When the user goes to your website, it downloads the HTML, JS and CSS and runs them inside the user's browser. That code is now unable to communicate with the server's operating system to have access to those environment variables.

There are a few techniques that are often used to go around this. You could switch your back-end server to a Node.js and create a route that would provide all the configuration which would be accessible by your front-end. But this requires us to change our base image from Nginx to Node.js and to create a web server. You could also use something like PHP or another server-side rendered language to change the HTML content before it gets served but once again, there requires a lot of changes in the way the application is built.

Thankfully, there is another way to do everything through a Dockerfile. We are already using Docker to initiate our build process. By adding a few commands, we can change those variables to something more generic in our build process. And then, just before our server starts, we can inject the environment variables into our configuration. This way, we don’t need to change anything in our development environment and we can keep our Nginx server for high performance and small footstep.  

To do so, you will need to put all your configuration in a single config.json file to make it easier to overwrite these settings. Next, you can use a bash script to overwrite them as part of the Docker build step. Finally, you can use `envsubst` to overwrite the environment variables with the actual values just before the server starts. If the environment variable changes and the container is restarted, it will take in those new values and send those to the end user.

In the next lesson, you will see how to do this in a somewhat naive approach. It will look for hard coded values such as "localhost:3000" and simply overwrite them by using `sed`, a Linux CLI tool to edit some text in a file. The new value of that JSON property will be the name of the environment variable, in this case "BASE_URL". This will let you use envsubst on the file to change the name of the variable into the actual value of that variable. This will work perfectly well in the context of this application and can work for you if you have a single value that needs to be stored in an environment variable.

The lesson after that will show you how to use `jq`, another CLI tool, to change the values inside the JSON file. This time, you won't rely on a hard-coded value but rather on the property of the JSON object. This will make for a more future-proof container but requires a little bit more work and some bash scripting. Feel free to skip that lesson if you don't want to dig deeper into this topic.

## Lesson script

In the last lesson, you’ve seen that all requests were made to http://localhost:3000. This means that this value was hard-coded somewhere in the front end source code. The first thing you will want to do is to find out where this was coded and make sure that it is only defined in one place. If you do a search for localhost:3000, you will see that it is only defined in one place. That’s good, half the work is done already. You will only need to modify the front/src/config.json file. But this value works well for our local development environment and maybe other developers on your team are using different values. For this reason, you will leave this value hard-coded in there, and you will change it as part of the build process in the Dockerfile.

To do so, you will use the a bash command. `sed` is a Linux utility that can be used to substitute some text in a file.

```docker
WORKDIR /app/src
RUN sed -i 's/http:\/\/localhost:3000/\$BASE_URL/' config.json
```

The result will be to change the config.json file to now have the following content.

```javascript
{
 "BASE_URL": "$BASE_URL"
}
```

This replacement works well when there is a single environment variable like in this example. However, if you want a solution that is more future-proof, it will be introduced in the next lesson.

Now that your configuration file has been changed to a generic value, you can run your npm install and build scripts. This will create all the minified CSS and JS files, with this generic value.

The next step will happen in the second step of our container. Just before the server starts, you will need to overwrite this $BASE_URL in your JavaScript package with the actual value of the BASE_URL environment variable. In order to do so, you will need to bypass the command that the Nginx base image uses to start the server with a different start script. This new script will use the envsubst command that is included with this image to substitute the $BASE_URL string with the actual value of it from the image environment variable. Now when you will start the container with a -e flag, it will take this value and overwrite the $BASE_URL string in the JavaScript file. Once this file is server to the browser, it will now contain the value from the environment variable.

To do so, you will write a small bash script.  You can call this file start_nginx.sh and place it in the /front folder. 

First, the script will run through all the app.*.js files and rename them to app.*.tmp.js. Then, for each of these .tmp.js file, you will use envsubst to replace the environment variable $BASE_URL with the actual value from the container environment variable. The output from the envsubst command then pipes to the original file name. Once all the JavaScript files have been modified, you can start the Nginx server with the "nginx -g" command.

```bash
#!/usr/bin/env bash
for file in /usr/share/nginx/html/js/app.*.js;
do
 if [ ! -f $file.tmp.js ]; then
   cp $file $file.tmp.js
 fi
 envsubst '$BASE_URL' < $file.tmp.js > $file
done
nginx -g 'daemon off;'
```

The final step as far as our container is concerned will be to copy this bash script into the container, change the permissions on it so that it can be executed and change the ENTRYPOINT of your container. An ENTRYPOINT is a command that is executed when the container is ready. It’s usually the command that "starts" your container. There can only be one ENTRYPOINT in a container so by adding it here, you are overriding the default ENTRYPOINT that was defined in the nginx base image.

```docker
...
COPY --from=builder /app/dist .
COPY start_nginx.sh /
RUN chmod +x /start_nginx.sh
ENTRYPOINT ["/start_nginx.sh"]
```

Your full Dockerfile should now look like this

```docker
# Dockerfile
FROM node:14 AS builder
COPY . /app
WORKDIR /app/src
RUN sed -i 's/http:\/\/localhost:3000/\$BASE_URL/' config.json
WORKDIR /app
RUN npm install
RUN npm run build
FROM nginx:1.17
WORKDIR /usr/share/nginx/html
COPY --from=builder /app/dist .
COPY start_nginx.sh /
RUN chmod +x /start_nginx.sh
ENTRYPOINT ["/start_nginx.sh"]
```

You are now ready to rebuild your front end image and start all three containers. To rebuild your front end, go to your /front folder and run the docker build command.

```bash
cd front
docker build -t k8scourse-front .
```

You can now restart all of your containers to see those new changes. First, stop all running containers. Next, go to your /db folder and start your database. Let the database start and run your back-end container. This time, you can run it on port 3001. And finally, start the front-end server and specify the BASE_URL for your backend.

```bash
docker stop k8scourse-db
docker stop back
docker stop front
cd ../db
docker run -p 3306:3306 -d --name k8scourse-db -e MYSQL_USER=user -e MYSQL_PASSWORD=mysql -e MYSQL_ROOT_PASSWORD=root --rm -v $(pwd)/init.sql:/docker-entrypoint-initdb.d/init.sql --network k8scourse mysql:5.7
docker run -p 3001:3000 --name back --rm --network k8scourse -e MYSQL_HOST=k8scourse-db -d k8scourse-back
docker run -p 8080:80 --name front --rm -e BASE_URL=http://localhost:3001 -d k8scourse-front
```

Now open up your browser and point it to [http://localhost:8080](http://localhost:8080). You should finally be able to try out the application. Everything is working again. And if you change any of the server ports or URLs, you will be able to use an environment variable to specify the new values and you won’t need to change your code. Also, those values such as the GIPHY API key won’t be in your repository and can be kept secret on your server. 

You are now almost ready to deploy those images to a server. Before you do so, you will have to ensure that these images run on any server. Right now, those containers are running as a root user inside their context. If you open up a bash shell and run the whoami command, you will see that you are running this session as the root user.

```bash
docker exec -it front /bin/bash
whoami
exit
```

You will need to do some small changes to your Dockerfiles to ensure that you are not running as root anymore. This will be done in the next lesson.
