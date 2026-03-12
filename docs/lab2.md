---
layout: default
title: 2. Update the app, build a new image
---

At this point, you should have a running todo list manager with a few items. Now, let's make a few changes and learn about managing our containers.

As a small feature request, we've been asked by the product team to change the "empty text" when we don't have any todo list items. They would like to transition it to the following:

    You have no todo items yet! Add one above!

Pretty simple, right? Let's make the change.


## Updating the source code

1. In the src/static/js/app.js file, update line 56 to use the new empty text.

    Change:

    ```
    <p className="text-center">No items yet! Add one above!</p>
    ```

    to:

    ```
    <p className="text-center">You have no todo items yet! Add one above!</p>
    ```

2. Build an updated version of the image, using the same command we used before.

    ```
    docker build -t todo-app .
    ```

3. Start a new container using the updated code.

    ```
    docker run -dp 3000:3000 todo-app
    ```

**Uh oh!** You probably saw an error like this (the IDs will be different):

    docker: Error response from daemon: driver failed programming external connectivity on endpoint laughing_burnell 
    (bb242b2ca4d67eba76e79474fb36bb5125708ebdabd7f45c8eaf16caaabde9dd): Bind for 0.0.0.0:3000 failed: port is already allocated.

So, what happened? Look at the complete message! As it happens so often, the important piece is at the end: "port is already allocated". We aren't able to start the new container because our old container is still running. The reason this is a problem is because that container is using the host's port 3000 and only one process on the machine (containers included) can listen to a specific port. To fix this, we need to remove the old container.

## Replacing the old container

To remove a container, it first needs to be stopped. Once it has stopped, it can be removed. 

1. Get the ID of the container by using the docker ps command.

    ```
    docker ps
    ```

2. Use the docker stop command to stop the container.

    ```
    docker stop <the-container-id>
    ```

3. Once the container has stopped, you can remove it by using the docker rm command.

    ```
    docker rm <the-container-id>
    ```

Be aware that the previous failed starting attempt left another container on your system. This container is "created" but not "started", the start failed. You can also have containers that are terminated by other means. 

`docker ps` only shows running containers. To show all (running, terminated, failed, etc) containers, add the '-a' flag:

```
docker ps -a
```

Result:

```
CONTAINER ID  IMAGE            COMMAND                 CREATED         STATUS         PORTS                   NAMES
8611edc94d23  todo-app         "docker-entrypoint.s…"  3 seconds ago   created                                quirky_jemison
```

You can remove this "dangling" container with the help of `docker rm <the-container-id>`.

### Tips

1. You can stop and remove a container in a single command by adding the "force" flag to the remove command:
    
    ```
    docker rm -f <the-container-id>
    ```

2. You can remove containers also using their name, either their assigned "funny" name (quirky_jemison, laughing_burnell, etc) or a name you specifically assigned to them. You will learn in a moment how to do that.

## Recap

While we were able to build an update, there were two things you might have noticed:

* All of the existing items in our todo list are gone! That's not a very good app! We'll talk about that in the next step.
* There were a lot of steps involved for such a small change. 

---

**Next Step:** [Persisiting the data, Volumes](lab3.md) 