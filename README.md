# docker-workshop

This repo is a workshop for introducting Docker.

## Use Case

We want to run a go http server inside Docker. The server runs on port 3000.

## Building

We know that we can compile the application using `make`, and that it compiles the binary to `dist/app`.

### Initial

So lets look at our initial Dockerfile:

```bash
# This is based on a golang docker image
FROM golang:1.11.0

# We are documenting that we are using the port 3000
EXPOSE 3000
```

### Attempt 1

We might start out by thinking, "*I can just build the binary locally and copy it into the container!*". This certainly works, but it requires discipline to ensure local builds are easily reproducible and documented for future team members.

We end up doing:

```bash
make && docker build . -t docker101
```

The Dockerfile might look like something like this:

```bash
FROM golang:1.11.0

# Does the src location into the dest location
COPY dist/app .

EXPOSE 3000

# By default the container will start app
CMD ["./app"]
```

### Attempt 2

[Attempt 1](#attempt-1) is an OK, approach. But what happens when we end up in a situation where a team member can't build the binary locally or CI server implodes? It would be really nice if we could build the go binary in complete isolation and all we need locally is the source code and Docker.

Since we are using the official golang Docker image and it comes with all the tools we need to build go binaries, why don't we give it a try:

```bash
FROM golang:1.11.0

# Set the path as an env var so we only need to set it in one place, could also be an ARG instruction instead
ENV SRC_DIR=/go/src/app

WORKDIR "${SRC_DIR}"

# Here we update the current packages, and upgrade installed ones, and create a /server dir.
RUN apt-get -y update
RUN apt-get -y upgrade
RUN mkdir -p /server

COPY . "${SRC_DIR}"

RUN make
RUN mv dist/app /server/app

EXPOSE 3000

WORKDIR /server

CMD ["./app"]
```

Let's build it:

```bash
docker build . -t docker101
```

Let's take a look at the image size:

```bash
docker images docker101
```

```bash
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
docker101           latest              c3436a3e237b        5 minutes ago       851MB
```

Now lets take a took at the layers:

```bash
docker history docker101
```

```bash
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
c3436a3e237b        4 minutes ago       /bin/sh -c #(nop)  CMD ["./app"]                0B                  
471b5ee11d7a        4 minutes ago       /bin/sh -c #(nop) WORKDIR /server               0B                  
2e626a79bbab        4 minutes ago       /bin/sh -c #(nop)  EXPOSE 3000                  0B                  
30891b7504d7        4 minutes ago       /bin/sh -c mv dist/app /server/app              6.49MB              
ac3f6e081d7e        4 minutes ago       /bin/sh -c make                                 27.9MB              
6890a5a6f2f2        4 minutes ago       /bin/sh -c #(nop) COPY dir:29c202c9f73978a91…   1.44kB              
bee7db56d3c9        4 minutes ago       /bin/sh -c mkdir -p /server                     0B                  
62cc7640be05        4 minutes ago       /bin/sh -c apt-get -y upgrade                   24.3MB              
8d2afa0b0da2        5 minutes ago       /bin/sh -c apt-get -y update                    16.3MB              
03be6ea25578        About an hour ago   /bin/sh -c #(nop) WORKDIR /go/src/app           0B                  
bfac6ed213bc        About an hour ago   /bin/sh -c #(nop)  ENV SRC_DIR=/go/src/app      0B                  
fb7a47d8605b        3 weeks ago         /bin/sh -c #(nop) WORKDIR /go                   0B                  
<missing>           3 weeks ago         /bin/sh -c mkdir -p "$GOPATH/src" "$GOPATH/b…   0B                  
<missing>           3 weeks ago         /bin/sh -c #(nop)  ENV PATH=/go/bin:/usr/loc…   0B                  
<missing>           3 weeks ago         /bin/sh -c #(nop)  ENV GOPATH=/go               0B                  
<missing>           3 weeks ago         /bin/sh -c set -eux;   dpkgArch="$(dpkg --pr…   341MB               
<missing>           3 weeks ago         /bin/sh -c #(nop)  ENV GOLANG_VERSION=1.11      0B                  
<missing>           3 weeks ago         /bin/sh -c apt-get update && apt-get install…   162MB               
<missing>           3 weeks ago         /bin/sh -c apt-get update && apt-get install…   142MB               
<missing>           3 weeks ago         /bin/sh -c set -ex;  if ! command -v gpg > /…   7.8MB               
<missing>           3 weeks ago         /bin/sh -c apt-get update && apt-get install…   23.1MB              
<missing>           3 weeks ago         /bin/sh -c #(nop)  CMD ["bash"]                 0B                  
<missing>           3 weeks ago         /bin/sh -c #(nop) ADD file:58d5c21fcabcf1eec…   101MB               
```

We can now see how each Docker instruction impacts the image size.

Now, let us apply some optimizations to see if we can reduce the size.

```bash
FROM golang:1.11.0

ENV SRC_DIR=/go/src/app

WORKDIR "${SRC_DIR}"

# Reduce the previous run operations into a single one.
RUN set -eux \
  && apt-get -y update \
  && apt-get -y upgrade \
  && apt-get -y clean \
  && rm -rf /var/lib/apt/lists/* \
  && mkdir -p /server

COPY . "${SRC_DIR}"

RUN set -eux \
  && make \
  && mv dist/app /server/app

EXPOSE 3000

WORKDIR /server

CMD ["./app"]

```

Let's rebuild:

```bash
docker build . -t docker101
```

```bash
docker images docker101
```

```bash
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
docker101           latest              45f30a069e32        5 seconds ago       829MB
```

```bash
docker history docker101
```

```
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
45f30a069e32        2 minutes ago       /bin/sh -c #(nop)  CMD ["./app"]                0B                  
4dd06c04b803        2 minutes ago       /bin/sh -c #(nop) WORKDIR /server               0B                  
e6ff3d853898        2 minutes ago       /bin/sh -c #(nop)  EXPOSE 3000                  0B                  
f723968a6e5a        2 minutes ago       /bin/sh -c set -eux   && make   && mv dist/a…   27.9MB              
c29663da4276        2 minutes ago       /bin/sh -c #(nop) COPY dir:7d837ccaa2fa622ba…   1.78kB              
971c5c37f11b        2 minutes ago       /bin/sh -c set -eux   && apt-get -y update  …   24.3MB              
03be6ea25578        About an hour ago   /bin/sh -c #(nop) WORKDIR /go/src/app           0B                  
bfac6ed213bc        About an hour ago   /bin/sh -c #(nop)  ENV SRC_DIR=/go/src/app      0B                  
fb7a47d8605b        3 weeks ago         /bin/sh -c #(nop) WORKDIR /go                   0B                  
<missing>           3 weeks ago         /bin/sh -c mkdir -p "$GOPATH/src" "$GOPATH/b…   0B                  
<missing>           3 weeks ago         /bin/sh -c #(nop)  ENV PATH=/go/bin:/usr/loc…   0B                  
<missing>           3 weeks ago         /bin/sh -c #(nop)  ENV GOPATH=/go               0B                  
<missing>           3 weeks ago         /bin/sh -c set -eux;   dpkgArch="$(dpkg --pr…   341MB               
<missing>           3 weeks ago         /bin/sh -c #(nop)  ENV GOLANG_VERSION=1.11      0B                  
<missing>           3 weeks ago         /bin/sh -c apt-get update && apt-get install…   162MB               
<missing>           3 weeks ago         /bin/sh -c apt-get update && apt-get install…   142MB               
<missing>           3 weeks ago         /bin/sh -c set -ex;  if ! command -v gpg > /…   7.8MB               
<missing>           3 weeks ago         /bin/sh -c apt-get update && apt-get install…   23.1MB              
<missing>           3 weeks ago         /bin/sh -c #(nop)  CMD ["bash"]                 0B                  
<missing>           3 weeks ago         /bin/sh -c #(nop) ADD file:58d5c21fcabcf1eec…   101MB   
```

We can see how reducing layers reduces disk size.

### Attempt 3

If we look at the golang image we are using, we can see the size alone is `776MB`!

```bash
docker images golang
```

```bash
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
golang              1.11.0              fb7a47d8605b        3 weeks ago         776MB
```

This seems quite large when our app makes up only a fraction of that amount... do we really need all of Debian in the first place? Do we need to keep all the build tooling after we built our binary?

What if we could use a lighter distro and copy the compiled binary to it?

Well we sure can!

We can take advantage of Docker multi-stage builds.

```bash
# We label this stage as go
FROM golang:1.11.0 as go

ENV SRC_DIR=/go/src/app

WORKDIR "${SRC_DIR}"

RUN set -eux \
  && apt-get -y update \
  && apt-get -y upgrade \
  && apt-get -y clean \
  && rm -rf /var/lib/apt/lists/* \
  && mkdir -p /server

COPY . "${SRC_DIR}"

RUN set -eux \
  && make \
  && mv dist/app /server/app

# We label this as an intermediate stage so we can easily clean up only intermediate images later
LABEL build.stage="intermediate"

###############################################################################
# We add another stage using the alpine linux image
FROM alpine:3.8

WORKDIR /server

# We can copy content from the go stage
COPY --from=go /server/app .

EXPOSE 3000

CMD ["./server/app"]

# We label this as the final stage for documentation purposes
LABEL build.stage="final"
```

Now let's see how much of an impact Alpine Linux has:

```bash
docker images docker101
```

```bash
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
docker101           latest              896e59dca90b        17 seconds ago      10.9MB
```

We've reduced the image from `~800+ MB` to only `~10 MB`. Amazing!

```bash
docker history docker101
```

```bash
IMAGE               CREATED              CREATED BY                                      SIZE                COMMENT
896e59dca90b        About a minute ago   /bin/sh -c #(nop)  LABEL build.stage=final      0B                  
b2f8246afd37        About a minute ago   /bin/sh -c #(nop)  CMD ["./server/app"]         0B                  
318025ecddf1        About a minute ago   /bin/sh -c #(nop)  EXPOSE 3000                  0B                  
c488b28e6d93        About a minute ago   /bin/sh -c #(nop) COPY file:ccdeca914c112520…   6.49MB              
fc54ea6511ec        7 minutes ago        /bin/sh -c #(nop) WORKDIR /server               0B                  
196d12cf6ab1        2 weeks ago          /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B                  
<missing>           2 weeks ago          /bin/sh -c #(nop) ADD file:25c10b1d1b41d46a1…   4.41MB  
```

We can see the difference in sizes between the layers. Alpine Linux just has less stuff compared to Debian, and works great when we our application compiles down to a binary.

## Tagging

Tags help us provide human readable identifiers for the Docker images we build and pull.

```bash
# build example

# push example

```

## Clean up

As we can see from our first few attempts at building Docker images, space can get eaten up really quick if you are not keeping watch!

```bash
# docker system prune

# docker image rm

# docker filters for multistage
```

## Volumes

## Ports

## Troubleshooting

```bash
# docker images

# docker inspect

# docker history
```
