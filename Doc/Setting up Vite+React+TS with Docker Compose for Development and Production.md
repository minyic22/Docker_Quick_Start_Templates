# Setting up Vite+React+TS with Docker Compose for Development and Production

## Introduction

Hi, fellow readers this blog is created just in case I forget how to setup the Vite project with Docker in the future and also help people who struggled with same issue.



# Prerequisites

If you are trying to follow this guild, I will assume you have the following installed on you OS already:

- React
- Vite
- Docker



## Procedures

### Step 1: Create a project with Vite

```
npm create vite@latest
```

![image-20230427211729927](C:\Users\minyi\Desktop\Docker\Doc\Setting up Vite+React+TS with Docker Compose for Development and Production.assets\image-20230427211729927.png)

You don't have to follow my setup, I just prefer to use TypeScript whenever possible.

The following image presents the files in the project created:

![image-20230427212305166](C:\Users\minyi\Desktop\Docker\Doc\Setting up Vite+React+TS with Docker Compose for Development and Production.assets\image-20230427212305166.png)

Now, edit `vite.config.ts` as below:

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [react()],
  server: {
    watch: {
      usePolling: true,
    },
    host: true, // needed for the Docker Container port mapping to work
    strictPort: true, // not necessary
    port: 3000, // you can replace this port with any port
  }
})
```

The `host` option is set to `true`, which is required for the Docker container port mapping to work correctly.

The `strictPort` option is set to `true`, which means that the application will only listen on the specified `port` and will not fall back to a different port if the specified port is unavailable.

The `port` option specifies that the application should listen on port 3000, which can be replaced with any other port number as needed.

### Step 2: Setup Dockerfile and .dockerignore

Now add a `Dockerfile` to the current working directory, and enter the following content:

```
FROM node as development

WORKDIR /usr/src/app

COPY package*.json .

RUN npm install

COPY . .

RUN npm run build

FROM nginx:stable-alpine as production

COPY --from=development /usr/src/app/dist /usr/share/nginx/html

COPY nginx.conf /etc/nginx/conf.d/default.conf

CMD ["nginx", "-g", "daemon off;"]
```

A Dockerfile is a set of instructions that specifies how to build a Docker image with all the necessary components to run an application. 

> Because Dockerfile has cache to commands that are already executed, hence it is important to put `COPY package*.json . ` before `RUN npm install` so `npm install` will not be executed every time the project is updated and copied to WORKDIR.
>
> ```
> COPY package*.json .
> 
> RUN npm install
> 
> COPY . . # copy from . project directory to . WORKDIR
> ```

It can have multiple stages thanks to Docker's layer structure, as demonstrated by the `FROM` keyword in this example, with stages for development and production.

> The first stage is `FROM node as development`, which means that we are using a Node.js image as the base image and calling it `development`. The subsequent commands in this stage will build an image that is suitable for development purposes.
>
> The `WORKDIR /usr/src/app` command sets the working directory for the subsequent commands to `/usr/src/app`.
>
> The `COPY package*.json .` command copies the package.json and package-lock.json files into the Docker image's working directory.
>
> The `RUN npm install` command installs the dependencies specified in the `package.json` file.
>
> The `COPY . .` command copies the entire application codebase into the Docker image's working directory.
>
> The `RUN npm run build` command builds the application by running the `build` script specified in the `package.json` file.
>
> The second stage is `FROM nginx:stable-alpine as production`, which means that we are using an Nginx image as the base image and calling it `production`. The subsequent commands in this stage will build an image that is suitable for production purposes.
>
> The `COPY --from=development /usr/src/app/dist /usr/share/nginx/html` command copies the built application files from the `development` stage to the `/usr/share/nginx/html` directory in the `production` stage.
>
> The `COPY nginx.conf /etc/nginx/conf.d/default.conf` command copies the `nginx.conf` file to the default Nginx configuration directory.
>
> The `CMD ["nginx", "-g", "daemon off;"]` command sets the command to run when a container is started from the image. In this case, it starts the Nginx server and runs it in the foreground, with the `daemon off;` option telling Nginx to run in the foreground.

 To ensure only the necessary files are present in production, this Dockerfile installs and builds only during development, copying the `dist/` folder to the production environment, which relies on a static web page served using Nginx as a reverse proxy.

Also add the following `.dockerignore` file so `node_modules` and `dist` does not get copied, because they will be created when `Dockerfile` is executed.



### Step 3: Create docker-compose.dev.yml and docker-compoes.yml for development and production

- **create `docker-compose.dev.yml` for development**

  ```
  version: '3.8'
  
  services:
    nao_drawing_board_dev:
      build:
        context: .
        target: development
      volumes:
        - .:/usr/src/app
        - /usr/src/app/node_modules
      ports:
        - 3000:3000
      command: npm run dev
  
  ```

  The above configuration file is written in YAML and is used to define a Docker service called `nao_drawing_board_dev`.

  The `build` section specifies that the Docker image for the service should be built using the `Dockerfile` in the current directory (`context: .`) and that the `development` stage should be used (`target: development`).

  The `volumes` section specifies two volumes to be mounted for the service: the current directory (`.`) to `/usr/src/app` inside the container, and the `node_modules` directory inside the container to prevent it from being overwritten by the mounted volume.

  The `ports` section specifies that port 3000 on the host machine should be mapped to port 3000 inside the container, which is where the `npm run dev` command is run.

  Overall, this `docker-compose.yml` configuration file sets up a development environment for a Node.js application in a container, with the source code mounted as a volume and the container exposing port 3000 for local development.

- create `docker-compose.yml` for production

  ```
  version: '3.8'
  
  services:
    api:
      build:
        context: .
        target: production
      ports:
        - "80:80"
  ```

  The above configuration file is written in YAML and is used to define a Docker service called `nao_drawing_board`.

  The `build` section specifies that the Docker image for the service should be built using the `Dockerfile` in the current directory (`context: .`) and that the `production` stage should be used (`target: production`).

  The `ports` section specifies that port 80 on the host machine should be mapped to port 80 inside the container.

  Overall, this `docker-compose.yml` configuration file sets up a production environment for a web application in a container, with the container exposing port 80 for external access.

`docker-compose.yml` is a configuration file that simplifies the process of defining and running multi-container Docker applications. It allows you to define the services, networks, and volumes required for your application, as well as their relationships and configurations, in one file. This makes it easy to manage complex multi-container applications and deploy them with a single command using `docker-compose`.



### Step 4: Create nginx.conf

As you may already notice, we have created a `nginx.conf` in the project directory, which will be copied to docker image. Now let us edit the `nginx.conf`  file:

```
server {
    listen       80;
    server_name  localhost;

    root /usr/share/nginx/html;

    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
}
```

It is important to configure the listen port, root directory and index file if it is named otherwise.

### Step 5: Build the docker image

#### Build development image:

```
docker compose -f docker-compose.dev.yml up --build
```

![image-20230427220800215](C:\Users\minyi\Desktop\Docker\Doc\Setting up Vite+React+TS with Docker Compose for Development and Production.assets\image-20230427220800215.png)

Now you should be able to access the development server that is running in the docker image from web browser in you host machine.

If no change is made to docker related files, you don't have to rebuild the image, you can just use the following:

```
docker compose -f docker-compose.dev.yml up
```

![image-20230427221245676](C:\Users\minyi\Desktop\Docker\Doc\Setting up Vite+React+TS with Docker Compose for Development and Production.assets\image-20230427221245676.png)

**Note that, if you change the src code in development mode, it will be reflected on the web page.**



#### Build production image:

```
docker compose up --build
```

![image-20230427221743631](C:\Users\minyi\Desktop\Docker\Doc\Setting up Vite+React+TS with Docker Compose for Development and Production.assets\image-20230427221743631.png)

Now you should be able to request the site from localhost:80(port 80 is default hence not revealed on image). 

Similarly, you don't have to rebuild the image if docker configuration is not changed:

```
docker compose up
```



## Conclusion

The full source code is uploaded on the following repo: https://github.com/minyic22/Docker_Quick_Start_Templates.

The next step will be uploaded the docker image with others for team development or pubic access.

Will update when I have time.