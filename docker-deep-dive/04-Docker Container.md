# Containers VS VMs

## VMs Architecture

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220426163226-vms-architecture.png)

## Containers Architecture

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220426163303-container-architecture.png)



# Container commands

- ***docker container run***** **is the command used to start new containers. In its simplest form, it accepts an image and a command as arguments. The image is used to create the container and the command is the application the container will run when it starts. This example will start an Ubuntu container in the foreground, and tell it to run the Bash shell: docker container run -it ubuntu /bin/bash. 

- ***Ctrl-PQ*** will detach your shell from the terminal of a container and leave the container running (UP) in the background.

-  ***docker container ls*** lists all containers in the running (UP) state. If you add the -a flag you will also see containers in the stopped (Exited) state.

- ***docker container exec*** runs a new process inside of a running container. It’s useful for attaching the shell of your Docker host to a terminal inside of a running container. This command will start a new Bash shell inside of a running container and connect to it: docker container exec -it <container-name or container-id> bash. For this to work, the image used to create the container must include the Bash shell.

- ***docker container stop*** will stop a running container and put it in the Exited (0) state. It does this by issuing a SIGTERM to the process with PID 1 inside of the container. If the process has not cleaned up and stopped within 10 seconds, a SIGKILL will be issued to forcibly stop the container. docker container stop accepts container IDs and container names as arguments.

- ***docker container start*** will restart a stopped (Exited) container. You can give docker container start the name or ID of a container.

- ***docker container rm*** will delete a stopped container. You can specify containers by name or ID. It is recommended that you stop a container with the docker container stop command before deleting it with docker container rm. 

- ***docker container inspect*** will show you detailed configuration and runtime information about a container. It accepts container names and container IDs as its main argument.

## Tips

1. "***-it***" represents interactive

1. Stopping containers gracefully. ***docker container stop is ***more polite than*** rm -rf . ***

1. 容器的自愈能力（Self-healing containers with restart policies）

    1. always

    1. unless-stopped

    1. on-failed

```text
$ docker container run --name neversaydie -it --restart always alpine sh
/#
```

