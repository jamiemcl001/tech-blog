---
layout: post
title:  "Communicating between Docker Containers, the Host Machine etc.!"
date:   2019-11-16 15:39:00 +0000
categories: docker
---
Hi all,

During our time at work we recently came across an issue when using LocalStack for testing. We had two containers spun up as part of a `docker-compose` stack, and needed to be able to communicate between the two containers. After spending some time looking through StackOverflow we realised where we were going wrong. Thus - the inspiration for this blog post (the first on my new tech blog 🎉) was born.

Rather than using this example throughout the post I'll simplify it even further to demonstrate. Say, for example, I have a simple Express server running on port 8080 of my local machine. When I make a GET request to the endpoint `/say-hello` with a name passed as a query parameter, I want to be nice and say hello to the fine person.

But what happens if I want to move the app over to execute on another host machine? Let's say that we have been developing and running the server on a *NIX based machine and we needed to migrate over to run this on a Windows machine? In this simplistic example we wouldn't expect to see any problems when we try run our app - but what if we were developing something much more complex than a simple server? We could easily run into issues that may only be encountered in a specific Operating System. So... Docker to the rescue. For anyone that hasn't played around with Docker as yet, it basically provides users with the ability to run commands/applications in a completely containerised environment - something that should make the different operating systems problem a thing of the past.

Listed above is just one of the many advantages of running our applications in a containerised environment - and if we start to run multiple containers alongside each other, we may want to be able to communicate between those - how do we do this? This article covers the following scenarios:

- **Communicating with a Docker container from the host machine**
- **Communicating with the host machine from inside a Docker container**
- **Communicating with another Docker container from inside a Docker container**

Let's begin...

### **Communicating with a Docker container from the host machine:**

If you clone [this repository](https://github.com/jamiemcl001/say-hello-server), and run the command `docker build -t sample-node .`, you will be able to run a container with this image. We will be using this image throughout the course of this blog post. Running the image in a newly constructed container is simple with the following command:

```shell
docker run -d -p 8080:8080 sample-node
```

The magic behind being able to connect to this container is the `-p 8080:8080` parameter. When used, this parameter will automatically map port 8080 of the docker container to port 8080 on your local machine. If you would rather use the standard port 80 to make requests then you could use the parameter, such as `-p 80:8080`. This will map port 80 of your localhost to port 8080 of the docker container.

With the previous commands being run - the local machine should be able to communicate directly with the docker host using the `localhost` name. Running `curl localhost:8080/say-hello?name=John` should result in you seeing `Hello, John` in the terminal. Hooray!

### **Communicating with the host machine from inside a Docker container:**

Unfortunately this way is much more tricky and not standard between different operating systems, although the use cases for it are more limited than, say, container <-> container communication and host machine -> container communication. Instructions for Windows machines will be added shortly _(unfortunately I don't readily have access to a Windows machine to test)_.

**Working with OSX**

Docker can automatically resolve the hostname `docker.for.mac.host.internal` when we run containers using the default bridge network. As such I can run the following command:

```
docker run -ti alpine /bin/sh
apk add curl
curl docker.for.mac.host.internal:8080/say-hello?name=John
```

If I want to do the same but with an easier to remember hostname (remembering to type out `docker.for.mac.host.internal` every time could become quite cumbersome then I can execute the following command to provide the hostname `local` instead.

```shell
docker run -ti --name sample_app --add-host local:$(ipconfig getifaddr en0) alpine /bin/sh
apk add curl
curl local:8080/say-hello?name=John
```

Both examples listed above should result in the response `Hello, John!` being shown in the terminal.

For context, the `--add-host` parameter will automatically add a record into the `/etc/hosts` file of the container. This essentially means that if you use the command `--add-host local:192.16.0.2`, then inspect the value of the hosts file - you should see something along the lines of the following:

```shell
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
192.16.0.2 local
172.17.0.3      3c6f479965ad
```

Therefore, when you make requests to `local` - the machine will first look in this file - find an associated record for the hostname `local` and therefore redirect the request to `192.16.0.2` without needing to go to the external network.

Please note that this is tested using OSX Catalina - unfortunately I don't have access to an earlier version of OSX to test if the `ipconfig getifaddr en0` works as expected, if it doesn't then please do let me know and I'll try and document a more reliable method. This command, when executed directly in the terminal will echo the local network's IP address for your machine. *There could be other network interfaces that you may need to check for in order to get this working. If anyone has a nicer way of retrieving the local IP of your machine then please mention them in the comments.*

**Working with Linux**

*Disclaimer: I've only tested this with an earlier version of Fedora (Fedora 27 to be exact) but I believe it should work on other Linux based OS'es. If anyone should encounter any issues then please let me know and I'll update the post with the appropriate method.*

Running Docker in Linux seems a little less fiddly than using it with OSX. The reason for this is that the `--network host` parameter works as you'd expect it to. If we begin by running our server on our local machine (`node src/server.js`), we can then easily allow our future containers to be able to communicate with it. Let's launch a new container using the super minimal `alpine` image.

```shell
docker run -ti --network host alpine
apk add curl
curl localhost:8080/say-hello?name=John
```

Now we should see the message `Hello John!` echoed back to us. See - much simpler than what we had to go through on OSX.

**Working with Windows**

Unfortunately I haven't had a chance to test the approaches required on Windows. Apologies for this - I'll check this shortly and update the post with the appropriate method.

### **Communicating with another Docker container from inside a Docker container:**

There are a few different ways of doing this, and I'll document my preferred approach: using a custom Docker network.

**Using a custom network:**

Lets begin by creating a new network in Docker. This post will not go into much detail about how networking works in Docker - although if we simply create a new network we can look at it as creating a sub-network that Docker will manage and allow containers to connect to.

```shell
docker network create private_network
```

With this command we have created a private bridge network named `private_network` in Docker. Containers that are attached to this network will be able to communicate with each other as if they are on their own private LAN. This is a really nice feature of Docker and something that I'll definitely be looking further into and discussing in the future.

```shell
docker run -d --name sample-node --network private_network sample-node
```

If we deconstruct this command - we are creating a container as we usually would do - but we are also adding the `--network private_network` parameter. This tells Docker to attach this newly running container to the private bridge network when it is launched. We can do this by running the command `docker network inspect private_network`

If you look at the output of this command, you should see a lot of the networks internals - but the thing that we are particularly interested in is the `Containers` object. This lists any containers that are currently attached to this internal network. At this point we should only see one container listed in this response - our newly created *sample-node* container.

```javascript
    "Containers": {
        "6ab48ccea7334c0dbc6469f5d0fa78ac9522ef2b806c25dd91beb22792143447": {
            "Name": "sample-node",
            "EndpointID": "...",
            "MacAddress": "...",
            "IPv4Address": "172.20.0.2/16",
            "IPv6Address": ""
        }
    },
```

This output essentially means that if any other containers should connect to the network - they will also be able to see others connected. We can demonstrate this by launching a throwaway container on the same network (`docker run -ti --rm --network private_network alpine /bin/sh`) and pinging `172.20.0.2`, seeing the following output:

```shell
PING 172.20.0.2 (172.20.0.2): 56 data bytes
64 bytes from 172.20.0.2: seq=0 ttl=64 time=0.320 ms
```

Docker also allows us to connect to other containers on the same network by their Container ID and their Container name. As a result, this makes it much easier for us to be able to communicate between them. This will mean that the docker bridge will resolve `sample-node` to `172.20.0.2`.

```shell
docker run -ti --network private_network alpine /bin/sh
apk add curl
curl sample-node:8080/say-hello?name=John
```

And we should see `Hello John` in our terminal output! 🎉🎉🎉

If users wish to use `docker-compose` instead of the Docker CLI to construct the containers then I'll also provide a `docker-compose.yml` file that can be used to demonstrate the same approach:

```docker
version: "3.7"

services:
    sample-node:
        image: sample-node
        container_name: sample-node
        networks:
            - default
    alpine:
        image: alpine
        container_name: alpinetest
        depends_on:
            - sample-node
        tty: true
    networks:
        - default

networks:
    default:
```

Running `docker-compose up -d` will start both of these containers and place them onto the same network on launch. Using the empty network object `default` empty will create a new bridge network automatically and attach both containers to it on launch, while using the `-d` parameter in docker-compose will launch all containers in detached mode. First off lets check that both containers are running as expected with the command `docker ps`. 

```shell
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS                  NAMES
e4a73515163c        alpine                 "/bin/sh"                7 hours ago         Up 7 hours                                 alpinetest
3e682b2adb74        sample-node            "node server.js"         7 hours ago         Up 7 hours          8080/tcp               sample-node
```

As we can tell from the output above, we have both containers running as expected, and we can move on to try executing commands within these containers. We can gain access to them by using the name we defined in `container_name` for each object. As such - lets gain access to the default shell in the `alpinetest` image, just to check that we can resolve the server as expected:

```shell
docker exec -ti alpinetest /bin/sh
apk add curl
curl sample-node:8080/say-hello?name=John
```

And we see our "Hello John" output, as expected! 🤯🤯🤯

By this point I feel that we've got a good enough understanding of how we can communicate between containers, as well as communicating with the host machine as well. This weekend has been eye opening to just how powerful Docker is and how we can use it for more things in our everyday work environment and I'll certainly be taking the time to learn more about it. I'll try and ensure that whenever something interesting does crop up - I'll try and write it up for future reference. I do hope that you found this article interesting and it has given you the opportunity to learn something new about Docker. I look forward to putting together the next blog post for this site.

Thanks,

Jamie.