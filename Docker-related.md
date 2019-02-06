### Setting up a local instance of a Docker container:

The following information largely pertains to Docker toolbox and older versions of Windows (i.e. Windows 7/8). Working on a VPN may cause problems for Docker machine, especially if LAN communication is disabled due to firewall. If LAN communication cannot be enabled or there is still a problem, a possible work around is to [forward the port as follows](https://www.iancollington.com/docker-and-cisco-anyconnect-vpn/ "forward the port as follows"):

1) `$docker-machine stop default`

2) `$export DOCKER_HOST="tcp://127.0.0.1:2376"`

3) `$"C:\Program Files\Oracle\VirtualBox\VBoxManage" modifyvm "default" --natpf1 "docker,tcp,,2376,,2376"`
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;$- Note that if you add VBoxManage to Path, you can avoid typing out the full path
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;$- This is used to configure port forwarding for the first network adapter

4) `$docker-machine start default`

At this stage, the machine may refuse the connection for security reasons. This can be disabled with:

5) `alias docker="docker --tlsverify=false"`

At this point, it should be possible to run a Docker container (e.g. $docker run hello-world).

When running containers with published ports, e.g.:

$docker run --name test -p 80:3000 testapp

a request on port[ 80 of the local machine should forward to port 3000](https://docs.docker.com/get-started/part2/#run-the-app " 80 of the local machine should forward to port 3000") of the Docker container. However, in the case of Windows 7/8, any attempts to access the Docker container should use the machine IP address, which can be found with:
`$docker-machine ip`

This is due to a [loopback problem](https://blog.sixeyed.com/published-ports-on-windows-containers-dont-do-loopback/ "loopback problem") with older versions of Windows.


----------------
### Building and running images:

While building images [with the Dockerfile](https://nodejs.org/en/docs/guides/nodejs-docker-webapp/ "with the Dockerfile"), there may be a need for [additional build resources](https://github.com/nodejs/docker-node/issues/282 "additional build resources"), such as python. As such, the following should be included:

```
RUN apk add --no-cache --virtual .gyp \
        python \
        make \
        g++ \
    && npm install \
        [ your npm dependencies here ] \
    && apk del .gyp
```

Moreover, if needed, multiple dockerfiles may be set up for different environments:

`$docker build -t test -f Dockerfile.dev .`

At this point the container can be spun up.

----------------
### Accessing/exploring images:

To access a running container, use the following:

`docker exec -it <container name> /bin/sh`

However, in the event that running the image causes an error and it exits immediately, exec will not be an option. Instead, the following allows for accessing of containers that fail to start properly.

Before attempting anything, check the available commands for the image using:

`$docker inspect [image]`

and make sure that bin/sh is available. In the event that an image is built from scratch, it may be absent. If present, use:

$docker run -\-rm -it -\-entrypoint=/bin/sh [image]

where -\-rm is for [clean up](https://docs.docker.com/engine/reference/run/#clean-up---rm) and -\-it is for interactive tty. This should allow for an interactive session, where the user can verify directories and other factors.

----------------------

Other useful commands:

Remove all exited containers: `docker rm ($docker ps -a -q)`

Remove all containers: `docker rm ($docker ps -a -q) -factors`

Remove all non-attached volumes: `docker volume prune`

Remove all images with <none> tag: `docker image rm $(docker images -f "dangling=true" -q)`
