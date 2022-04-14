![](https://tcs.teambition.net/storage/312g4288f43d970570c8d3a55db5cbcc91d6?Signature=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBcHBJRCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9hcHBJZCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9vcmdhbml6YXRpb25JZCI6IiIsImV4cCI6MTY1MDUzNDM0NywiaWF0IjoxNjQ5OTI5NTQ3LCJyZXNvdXJjZSI6Ii9zdG9yYWdlLzMxMmc0Mjg4ZjQzZDk3MDU3MGM4ZDNhNTVkYjVjYmNjOTFkNiJ9.yt3GloNtYGBrImAZFk3mqkl78HhG6i7uGQdd5HmY-nU&download=image.png "")

Containers are all about making apps simple to build, ship, and run. The process of containerizing an app looks like this:

1. Start with your application code and dependencies

1. Create a Dockerfile that describes your app, its dependencies, and how to run it

1. Feed the Dockerfile into the docker image build command

1. Push the new image to a registry (optional)

1. Run container from the image

Once your app is containerized (made into a container image), you’re ready to share it and run it as a container.

# Dockerfile

```text
FROM alpine
LABEL maintainer="nigelpoulton@hotmail.com"
RUN apk add --update nodejs nodejs-npm
COPY . /src
WORKDIR /src
RUN npm install
EXPOSE 8080
ENTRYPOINT ["node", "./app.js"]
```

# Steps

## build image

 `docker image build -t web:latest .`

## tag image

 `docker image tag web:latest nigelpoulton/web:latest`

## push image 

You need to try ***docker login*** first.

`docker image push nigelpoulton/web:latest`

## run image

`docker container run -d --name c1 -p 80:8080 web:latest`

# Analysis

## image history

```shell
$ docker image history web:latest
IMAGE CREATED BY SIZE
fc6..18e /bin/sh -c #(nop) ENTRYPOINT ["node" "./a... 0B
334..bf0 /bin/sh -c #(nop) EXPOSE 8080/tcp 0B
b27..eae /bin/sh -c npm install 14.1MB
932..749 /bin/sh -c #(nop) WORKDIR /src 0B
052..2dc /bin/sh -c #(nop) COPY dir:2a6ed1703749e80... 22.5kB
c1d..81f /bin/sh -c apk add --update nodejs nodejs-npm 46.1MB
336..b92 /bin/sh -c #(nop) LABEL maintainer=nigelp... 0B
3fd..f02 /bin/sh -c #(nop) CMD ["/bin/sh"] 0B
<missing> /bin/sh -c #(nop) ADD file:093f0723fa46f6c... 4.15MB
```

## inspect image

```shell
$ docker image inspect web:latest
<Snip>
},
"RootFS": {
"Type": "layers",
"Layers": [
"sha256:cd7100...1882bd56d263e02b6215",
"sha256:b3f88e...cae0e290980576e24885",
"sha256:3cfa21...cc819ef5e3246ec4fe16",
"sha256:4408b4...d52c731ba0b205392567"
]
},

```

# Multi Stage

```shell
FROM node:latest AS storefront
WORKDIR /usr/src/atsea/app/react-app
COPY react-app .
RUN npm install
RUN npm run build

FROM maven:latest AS appserver
WORKDIR /usr/src/atsea
COPY pom.xml .
RUN mvn -B -f pom.xml -s /usr/share/maven/ref/settings-docker.xml dependency:resolve
COPY . .
RUN mvn -B -s /usr/share/maven/ref/settings-docker.xml package -DskipTests

FROM java:8-jdk-alpine AS production
RUN adduser -Dh /home/gordon gordon
WORKDIR /static
COPY --from=storefront /usr/src/atsea/app/react-app/build/ .
WORKDIR /app
COPY --from=appserver /usr/src/atsea/target/AtSea-0.0.1-SNAPSHOT.jar .
ENTRYPOINT ["java", "-jar", "/app/AtSea-0.0.1-SNAPSHOT.jar"]
CMD ["--spring.profiles.active=postgres"]

```

```shell
$ docker image ls
REPO TAG IMAGE ID CREATED SIZE
node latest a5a6a9c32877 5 days ago 941MB
<none> <none> d2ab20c11203 9 mins ago 1.11GB
maven latest 45d27d110099 9 days ago 508MB
<none> <none> fa26694f57cb 7 mins ago 649MB
java 8-jdk-alpine 3fd9dd82815c 7 months ago 145MB
multi stage 3dc0d5e6223e 1 min ago 210MB

```

