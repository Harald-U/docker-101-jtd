---
layout: default
title: 5. Image Building Best Practises
---

Here is some useful information and best practises for Docker Images and Image Building.

<!-- ## Security Scanning

When you have built an image, it is good practice to scan it for security vulnerabilities using the docker scan command. Docker has partnered with Snyk to provide the vulnerability scanning service.

**Note:** Unfortunately, this feature is only available for registered users, you need to logon with your Docker ID first.

For example, to scan the todo-app image you created earlier in the tutorial, you can just type

```
docker scan todo-app
```

The scan uses a constantly updated database of vulnerabilities, so the output you see will vary as new vulnerabilities are discovered, but it might look something like this:

```
Testing todo-app...

Tested 177 dependencies for known vulnerabilities, found 1 vulnerability.


Issues with no direct upgrade or patch:
  ✗ Regular Expression Denial of Service (ReDoS) [Low Severity][https://security.snyk.io/vuln/SNYK-JS-DEBUG-3227433] in debug@2.6.9
    introduced by express@4.18.2 > debug@2.6.9 and 4 other path(s)
  This issue was fixed in versions: 3.1.0
```

The output lists the type of vulnerability, a URL to learn more, and importantly which version of the relevant library fixes the vulnerability.

There are several other options, which you can read about in the [docker scan documentation](https://docs.docker.com/engine/scan/).
-->

## Image Layering

* A Docker image is built up from a series of layers. 
* Each layer represents an instruction in the image’s Dockerfile. 
* Each layer except the very last one is read-only. 
* Each layer is only a set of differences from the layer before it. Note that both adding, and removing files will result in a new layer.
* The layers are stacked on top of each other. 
* A method called [union mounting](https://en.wikipedia.org/wiki/Union_mount){:target="_blank"} is used to combine these layers into one filesystem.

You can look at what makes up an image using the `docker image history` command. You can see the instruction that was used to create each layer within an image.

Use the `docker image history` command to see the layers in the todo-app image you created earlier in the tutorial.

```
docker image history todo-app
```

You should get output that looks something like this (dates/IDs may be different).

```
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
2f00d62f2528   24 minutes ago   CMD ["node" "src/index.js"]                     0B        buildkit.dockerfile.v0
<missing>      24 minutes ago   EXPOSE map[3000/tcp:{}]                         0B        buildkit.dockerfile.v0
<missing>      24 minutes ago   RUN /bin/sh -c yarn install --production # b…   85.3MB    buildkit.dockerfile.v0
<missing>      24 minutes ago   COPY . . # buildkit                             4.59MB    buildkit.dockerfile.v0
<missing>      24 minutes ago   WORKDIR /app                                    0B        buildkit.dockerfile.v0
<missing>      8 weeks ago      /bin/sh -c #(nop)  CMD ["node"]                 0B        
<missing>      8 weeks ago      /bin/sh -c #(nop)  ENTRYPOINT ["docker-entry…   0B        
<missing>      8 weeks ago      /bin/sh -c #(nop) COPY file:4d192565a7220e13…   388B      
<missing>      8 weeks ago      /bin/sh -c apk add --no-cache --virtual .bui…   7.77MB    
<missing>      8 weeks ago      /bin/sh -c #(nop)  ENV YARN_VERSION=1.22.19     0B        
<missing>      8 weeks ago      /bin/sh -c addgroup -g 1000 node     && addu…   117MB     
<missing>      8 weeks ago      /bin/sh -c #(nop)  ENV NODE_VERSION=18.19.0     0B        
<missing>      2 months ago     /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B        
<missing>      2 months ago     /bin/sh -c #(nop) ADD file:1f4eb46669b5b6275…   7.38MB    
```

Each of the lines represents a layer in the image. The display here shows the layers that are part of the base image (`FROM node:18-alpine`) at the bottom (the lines where IMAGE is missing and that are created 2 months or 8 weeks ago) and the newest layer at the top (created 24 minutes ago). Using this, you can also quickly see the size of each layer, helping diagnose large images.

## Layer Caching

Now that you've seen the layering in action, there's an important lesson to learn to help decrease build times for your container images.

> Once a layer changes, all downstream layers have to be recreated as well

Let's look at the Dockerfile we were using one more time...

```
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN yarn install --production
EXPOSE 3000
CMD ["node", "src/index.js"]
```

Going back to the image history output, we see that **each command in the Dockerfile becomes a new layer in the image**. 

You might remember that when we made a very small change to the image (we simply changed a string in one file), the Node.js dependencies had to be reinstalled which takes a long time. Is there a way to fix this? 

To fix this, we need to restructure our Dockerfile to help support the caching of the dependencies. For Node-based applications, those dependencies are defined in the package.json file. So, what if we copied only that file in first, install the dependencies, and then copy in everything else? Then, we only recreate the yarn dependencies if there was a change to the package.json. Make sense?

1. Update the Dockerfile to copy in the `package.json` (and yarn.lock) first, install dependencies, and then copy everything else in.

    ```
    FROM node:18-alpine
    WORKDIR /app
    COPY package.json yarn.lock ./
    RUN yarn install --production
    EXPOSE 3000
    COPY . .
    CMD ["node", "src/index.js"]
    ```

2. Create a file named `.dockerignore` in the same folder as the Dockerfile with the following contents:

   ```
   node_modules
   ```

    `.dockerignore` files are an easy way to selectively copy only image relevant files. You can read more about this [here](https://docs.docker.com/engine/reference/builder/#dockerignore-file){:target="_blank"}. In this case, the local `node_modules` folder should be omitted in the second `COPY` step because otherwise, it would possibly overwrite files which were created by the command in the RUN step. 
    
    If you develop in Python, I found this [blog](https://www.docker.com/blog/containerized-python-development-part-1/){:target="_blank"} for you.

3. Build a new image.

    ```
    docker build -t todo-app .
    ```

    You should see output like this...

        ```
        [+] Building 9.0s (10/10) FINISHED                                                                             docker:default
        => [internal] load build definition from Dockerfile                                                           0.0s
        => => transferring dockerfile: 183B                                                                           0.0s
        => [internal] load metadata for docker.io/library/node:18-alpine                                              0.0s
        => [internal] load .dockerignore                                                                              0.0s
        => => transferring context: 54B                                                                               0.0s
        => [1/5] FROM docker.io/library/node:18-alpine                                                                0.0s
        => [internal] load build context                                                                              0.1s
        => => transferring context: 2.85kB                                                                            0.1s
        => CACHED [2/5] WORKDIR /app                                                                                  0.0s
        => [3/5] COPY package.json yarn.lock ./                                                                       0.0s
        => [4/5] RUN yarn install --production                                                                        8.2s
        => [5/5] COPY . .                                                                                             0.1s 
        => exporting to image                                                                                         0.6s 
        => => exporting layers                                                                                        0.6s 
        => => writing image sha256:c150598f61853d4c86c423adcc920f655b23051560a5c3800c63c074d45aa6d5                   0.0s 
        => => naming to docker.io/library/todo-app                                   
        ```

    You'll see that most layers were rebuilt. Perfectly fine since we changed the Dockerfile quite a bit. 

4. Now, make a change to the `src/static/index.html` file (e.g. change the `title` to say "The Awesome Todo App").

5. Rebuild the Docker image again using `docker build -t todo-app .` again. This time, your output should look a little different.

        ```
        [+] Building 0.2s (10/10) FINISHED                                                                             docker:default
        => [internal] load build definition from Dockerfile                                                           0.0s
        => => transferring dockerfile: 183B                                                                           0.0s
        => [internal] load metadata for docker.io/library/node:18-alpine                                              0.0s
        => [internal] load .dockerignore                                                                              0.0s
        => => transferring context: 54B                                                                               0.0s
        => [1/5] FROM docker.io/library/node:18-alpine                                                                0.0s
        => [internal] load build context                                                                              0.1s
        => => transferring context: 3.48kB                                                                            0.1s
        => CACHED [2/5] WORKDIR /app                                                                                  0.0s
        => CACHED [3/5] COPY package.json yarn.lock ./                                                                0.0s
        => CACHED [4/5] RUN yarn install --production                                                                 0.0s
        => [5/5] COPY . .                                                                                             0.0s
        => exporting to image                                                                                         0.0s
        => => exporting layers                                                                                        0.0s
        => => writing image sha256:b67d89be99c1317c4355440b51cdd28697998c5b0335d6396f0db01c30235674                   0.0s
        => => naming to docker.io/library/todo-app        
        ```

    First off, you should notice that the build was MUCH faster (0.2 s vs. 9.0 s)! And, you'll see that steps 2,3, and 4 all have been CACHED. So we are using the build cache. Pushing and pulling this image and updates to it will be much faster as well. 

## "Dangling" Images, Image Tags

During this lab, whenever we build and rebuild the todo-app container image, we did so without specifiying a tag with the name.

List all container images with this command:

```
docker image ls
```

and check for the todo image:

```
REPOSITORY                    TAG         IMAGE ID       CREATED              SIZE
todo-app                      latest      6866f3c2eb2e   34 seconds ago       222MB
<none>                        <none>      9c60391389f0   About a minute ago   222MB
<none>                        <none>      7c8c52d18f97   19 minutes ago       222MB
<none>                        <none>      53c0e6d51d54   36 minutes ago       222MB
...
```

When you build a container image with a name but without a tag, Docker will give it the tag "latest".

Note the three images with name and tag of `<none>`. Whenever you build a new todo-app image, it replaces the prevously built image which in turn "looses" the name and the tag.

In a real scenario, whenever you build a container, you should give it a meaningful tag, e.g. v1 for Version 1:

```
docker build -t todo-app:v1 .
```

And when you add functions, use a new tag to build a new image:

```
docker build -t todo-app:v2 .
```

To get rid of the "dangling" images (the `<none> <none>` images) use this command:

```
docker image prune
```

## Multi-Stage Builds

While we are not going to dive into it too much in this tutorial, multi-stage builds are an incredibly powerful tool to help use multiple stages to create an image. There are several advantages for them:

* Separate build-time dependencies from runtime dependencies
* Reduce overall image size by shipping only what your app needs to run

### Maven/Tomcat Example

When building Java-based applications, a JDK is needed to compile the source code to Java bytecode. However, that JDK isn't needed in production. Also, you might be using tools like Maven or Gradle to help build the app. Those also aren't needed in our final image. Multi-stage builds help.

```
FROM maven AS build
WORKDIR /app
COPY . .
RUN mvn package

FROM tomcat
COPY --from=build /app/target/file.war /usr/local/tomcat/webapps 
```

In this example, we use one stage (called `build`) to perform the actual Java build using Maven. In the second stage (starting at `FROM tomcat`), we copy in files from the build stage. The final image is only the last stage being created (which can be overridden using the `--target` flag).

### React Example

When building React applications, we need a Node environment to compile the JS code (typically JSX), SASS stylesheets, and more into static HTML, JS, and CSS. If we aren't doing server-side rendering, we don't even need a Node environment for our production build. Why not ship the static resources in a static nginx container?

```
FROM node:12 AS build
WORKDIR /app
COPY package* yarn.lock ./
RUN yarn install
COPY public ./public
COPY src ./src
RUN yarn run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
```

Here, we are using a `node:12` image to perform the build (maximizing layer caching) and then copying the output into an nginx container. 

## Recap

By understanding a little bit about how images are structured, we can build images faster and ship fewer changes. Multi-stage builds help us reduce overall image size and increase final container security by separating build-time dependencies from runtime dependencies. 



---

**Next Step:** [Docker Compose](lab6.md)