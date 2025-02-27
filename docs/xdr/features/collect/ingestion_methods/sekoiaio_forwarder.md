# Sekoia.io Forwarder

## Overview

[Docker](https://docs.docker.com/get-started/overview/) is a tool that can be used to run packaged applications in an isolated environment on a host.
Packaged applications are stored in an object called an image, which includes an OS, the dependencies and the configuration. With that, the application will have the same behaviour whatever the OS used on the host as long as it's a x86-64 Linux host.

Sekoia.io offers a preconfigured concentrator based on Docker to forward events on the platform.

This method simplifies as much as possible the configuration needed to set up a concentrator in order to collect logs and send them on each relevant Intakes.

!!! Warning
    In this method, each technology MUST send their logs on different ports (of your choice) of the concentrator in order to make it work. The main principle of this method is to discriminate each technology by port to link them with the right Intake key.

Please find our English tutorial video below to see how to configure the forwarder ! French version is also available [here](https://youtu.be/CSEG2flmffE){:target="_blank"}.
[![French tutorial](https://img.youtube.com/vi/CSEG2flmffE/0.jpg)](https://youtu.be/BwydQiWMlv0){:target="_blank"}

## Prerequisites

* A x86-64 Linux host
* Last version of Docker Engine. Please follow [this section](#docker-engine-installation) to install it if needed
* INBOUND TCP or UDP flows opened between the systems/applications and the concentrator on the ports of your choice
* OUTBOUND TCP flow opened towards intake.sekoia.io on port 10514

## Configure your concentrator

Two files are needed to run the concentrator: `docker-compose.yml` and `intakes.yaml`.

1. Create a folder where the configuration and data of the concentrator will be stored

    ```bash
    mkdir sekoiaio-concentrator && cd sekoiaio-concentrator
    ```

2. Create the two files

    ```bash
    touch docker-compose.yml && touch intakes.yaml
    ```

### Configure intakes.yaml file

The `intakes.yaml` file is used to tell the concentrator how to bind a port where logs are received to its technology represented by the Intake key.
For each technology, specify:

* a name: it has nothing to do with Sekoia.io, feel free to use the explicite value of your choice
* the protocol: `tcp` or `udp`
* a port: to process incoming events
* the Intake key: can be retreived from the Intakes page of your community

Here is an example of the file with 3 technologies sending their events to the concentrator on ports `20516`, `20517` and `20518`. Feel free to copy paste it and adapt it to your need:

```yaml
---
intakes:
- name: Techno1
  protocol: tcp
  port: 20516
  intake_key: INTAKE_KEY_FOR_TECHNO_1
- name: Techno2
  protocol: tcp
  port: 20517
  intake_key: INTAKE_KEY_FOR_TECHNO_2
- name: Techno3
  protocol: tcp
  port: 20518
  intake_key: INTAKE_KEY_FOR_TECHNO_3
```

!!! Tip
    You are not limited to 3 entries. Feel free to adapt it to your needs.

### Configure docker-compose.yml file

Please find below a template of the `docker-compose.yml` file.

```yaml
version: "3.9"
services:
  sekoia_concentrator:
    logging:
      options:
        max-size: "1000m"
        max-file: "2"
    image: ghcr.io/sekoia-io/sekoiaio-docker-concentrator:0.9
    environment:
      - MEMORY_MESSAGES=100000
      - DISK_SPACE=32g
    ports:
      - "20516-20518:20516-20518"
    volumes:
      - ./intakes.yaml:/intakes.yaml
      - ./conf:/etc/rsyslog.d
      - ./disk_queue:/var/spool/rsyslog
    restart: always
    pull_policy: always
```

!!! Note
    Feel free to copy paste it and adapt it to your needs.

The following sections will describe each element of this file above.

#### Logging

```yaml
logging:
    options:
    max-size: "1000m"
    max-file: "2"
```

Docker logging system offers the flexibility to view events received on the container in real time with the command `docker logs <container_name>`. These logs are stored by default in `/var/lib/docker/containers/<container_uuid>/<container_uuid>-json.log`. 

To avoid the overload of disk space on your host, some options are specified. `max-size` specifies the maximum size of one file and `max-file` specifies the total number of files allowed. When the maximum number of files is reached, a log rotation is performed and the oldest file is deleted.

!!! Note
    Docker logging system is an independent solution from Sekoia.io or the buffer you want to set up on your concentrator. It is only used to view the last events on your concentrator.

#### Environment variables

```yaml
environment:
    - MEMORY_MESSAGES=100000
    - DISK_SPACE=32g
```

Two environment variables are used to customize the container. These variables are used to define a queue for incoming logs in case there is a temporarily issue in transmitting events to Sekoia.io. The queue stores messages in memory up to a certain number of events and then store them on disk. When the issue is fixed, events stored are retrieved from the queue and forward to the plateform.

* `MEMORY_MESSAGES=100000` means the queue is allowed to store up to 100,000 messages in memory. Since in the image configuration the maximum value of a message is 20ko, 100,000 means 100,000 * 20,000 = 2Go 
* `DISK_SPACE=32g` means the queue is allowed to store on disk up to 32 giga of messages.

#### Ports

```yaml
ports:
    - "20516-20518:20516-20518"
```

As specified in the Overview section, the concentrator will be run in an isolated environment. That means, by default, no flow is open between the host and the concentrator. 
`20516-20518:20516-20518` means that every packets coming through the TCP port `20516`, `20517` or `20518` to the host will be forwarded to the concentrator container on the port 20516, 20517 or 20518.

If you want to open a UDP flow, please add a line with `/udp` at the end. For instance, to open the TCP flows `20516`, `20517`, `20518` and the UDP flow `20519`, the ports section will be:

```yaml
ports:
    - "20516-20518:20516-20518"
    - "20519:20519/udp"
```

!!! Warning
    Please adapt these values according to the intakes.yaml file.

#### Volumes

```yaml
volumes:
    - ./intakes.yaml:/intakes.yaml
    - ./conf:/etc/rsyslog.d
    - ./rsyslog:/var/spool/rsyslog
```

Volumes are used to share files and folders between the host and the container:

* `./intakes.yaml:/intakes.yaml` is used to tell the concentrator what protocol, ports and intake keys to use.
* `./conf:/etc/rsyslog.d` is mapped if you want to customize some rsyslog configuration (ADVANCED)
* `./disk_queue:/var/spool/rsyslog` is used when the concentrator queue stores data on disk. The mapping avoids data loss if logs are stored on disk and the container is deleted.

#### Additional options

```yaml
restart: always
pull_policy: always
```

* `restart: always`: this line indicates to restart the concentrator everytime it stops. That means if it crashes, if you restart Docker or if you restart the host, the concentrator will start automatically.
* `pull_policy: always`: docker compose will always try to pull the image from the registry and check if a new version is available for the tag specified.

## Start the concentrator

To start the concentrator, run the following command in the folder where `docker-compose.yml` and `intakes.yaml` are stored:

```bash
sudo docker compose up -d
```

You should have an answer like this:

```bash
[+] Running 1/1
 ⠿ Container docker-compose-rsyslog-1  Started
```

!!! Note
    The first time you start the container, you will download the image stored on the GitHub repository - `ghcr.io/sekoia-io/sekoiaio-docker-concentrator`

To check the status of the concentrator, you can run:

```bash
sudo docker compose ps
```

## Useful commands

**To view all the concentrator logs:**

```bash
sudo docker compose logs
```

Everytime an event is sent to the concentrator, the event is shown in these logs.

**To view last logs in real time:**

```bash
sudo docker compose logs -f
```

**To stop the concentrator:**

```bash
sudo docker compose stop
```

**To delete the concentrator (the concentrator needs to be stopped):**

```bash
sudo docker compose rm
```

## Troubleshooting

You can't find the logs in your community? No worries this section will give you an advice to identify what is happening.

### Step 1: check if the events are received by the concentrator

To check if the events are received by the concentrator, you can run the following command that will display all the last logs received:

```bash
sudo docker compose logs
```

If you are looking for specific logs, you can also run:

```bash
sudo docker compose logs | grep "YOUR_PATTERN"
```

Finally, if you want to check if events are coming in real time:

```bash
sudo docker compose logs -f
```

**You don't see your events with these commands?** 

1. Check the `intakes.yaml` file to see if you have declared the protocols and ports you wanted. 

2. Verify if this information is taken into account by the concentrator. At start-up, the concentrator always shows the list of Intakes with the protocols and ports.
    ```bash
    sudo docker compose logs | more
    ```

3. Check that you correctly declared the `ports` section in the `docker-compose.yml` file. They MUST be the same as the ports declared in the `intakes.yaml` file. For instance, if you declared 4 technologies on ports `25020`, `25021`, `25022` and `25023`, the ports line the `docker-compose.yml` has to be `"25020-25023:25020-25023"`. 

4. Verify that traffic is incoming from your log source, meaning no firewall is blocking the events.
    ```bash
    sudo tcpdump -i <change_with_interface_name> -c10 -nn src <remote_ip> -vv
    ```

    To find those values:

    - `change_with_interface_name`use the command `ip addr`
    - `remote_ip`is the IP from which the logs should be incoming 

### Step 2: verify everything is correctly configured to forward events to Sekoia.io

1. Check the Intake keys you wrote in `intakes.yaml` are correct.

2. Check the network flow between the concentrator host and Sekoia.io is opened to the destination `intake.sekoia.io` on protocol `TCP` and port `10514`. You can easily check it with `telnet`:
    ```bash
    sudo apt install telnet && telnet intake.sekoia.io 10514
    ```

    This command should return the following answer, meaning the network flow is opened:

    ```bash
    Connected to intake.sekoia.io.
    Escape character is '^]'.
    ```

    Remove `telnet` if don't need it anymore

    ```bash
    sudo apt remove telnet
    ```

3. Finally check the status of the Sekoia.io plateform on [https://status.sekoia.io](https://status.sekoia.io).

## Additional information

The image used to run the concentrator is maintained on [this github repository](https://github.com/SEKOIA-IO/sekoiaio-docker-concentrator). Feel free to contribute and make a pull request to improve the concentrator!

## How to update the concentrator
Docker uses the notion of tag to identify the version of an image. The tag is always referenced in line starting with `image` in `docker-compose.yml`:

```
image: ghcr.io/sekoia-io/sekoiaio-docker-concentrator:0.9
```

`0.9` means the version used by `docker compose` is 0.9. You can find all the versions available on the GitHub repository [here](https://github.com/SEKOIA-IO/sekoiaio-docker-concentrator/pkgs/container/sekoiaio-docker-concentrator/versions?filters%5Bversion_type%5D=tagged)

To update the concentrator, just change the tag in `docker-compose.yml`, then recreate the concentrator with the command:
```bash
sudo docker compose up -d
```

There is a special tag called `latest` which always refers to the latest version. If you use this tag, every time the concentrator will be recreated, the image will be updated to the latest version.

!!! Important
    Be very careful if you use the tag `latest` in production! If a major change is present in a new version, it could break your current concentrator with the settings in place.

!!! Important
    Each image version is also updated every week in order to update the OS contained in the image with the latest security patches. You are responsible for recreating the concentrator periodically to initiate the image update.


## How do I add a new Intake later?
To add a new Intake, it's very simple ! Follow these steps:

1. Edit the `intakes.yaml` file and add the new intake
    ```yaml
    - name: NewIntake
      protocol: tcp
      port: 20519
      intake_key: INTAKE_KEY_FOR_NEW_INTAKE
    ```
2. Edit the `docker-compose.yml` file and modify the `ports` section according to the `intakes.yaml` file
    ```yaml
    ports:
    - "20516-20519:20516-20519"
    ```
3. Recreate the concentrator by running the command:
    ```bash
    sudo docker compose up -d
    ```
4. Check in the concentrator logs that the new Intake is taken into account
    ```bash
    sudo docker compose logs | more
    ```

## Docker Engine installation

This section describes how to install Docker using the `apt` repository on one of those Debian 64-bit versions:

* Debian Bullseye 11 (stable)
* Debian Buster 10 (oldstable)

All of the installation steps come from the [official Docker Engine installation for Debian](https://docs.docker.com/engine/install/debian/). Feel free to consult the official installation page for more information.
To install Docker on another Linux OS, please consult the [official Docker documentation](https://docs.docker.com/engine/install/)

### Uninstall old versions (if applicable)

Run the following command to uninstall old Docker versions:

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```

### Set up the repository

1. Update the `apt` package index and install packages to allow `apt` to use a repository over HTTPS

    ```bash
    sudo apt-get update
    sudo apt-get install \
        ca-certificates \
        curl \
        gnupg \
        lsb-release
    ```

2. Add Docker's official GPG key

    ```bash
    sudo mkdir -m 0755 -p /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    ```

3. Use the following command to set up the repository

    ```bash
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
      $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```

### Install Docker Engine

1. Update the `apt` package index

    ```bash
    sudo apt-get update
    ```

2. Install Docker Engine, containerd, and Docker compose

    ```bash
    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```

3. Verify that the Docker Engine installation is successful by running the hello-world image

    ```bash
    sudo docker run hello-world
    ```

## 5 minutes setup on Debian

!!! Warning
    This script will automate all the steps detailed on this page **for a Debian host**. Please read  the content of this page carefully before executing it.

Connect to the remote server where you would like to install the Sekoia.io Forwarder, then follow those steps:

1. Execute a script to setup the docker

    ```bash
    wget https://raw.githubusercontent.com/SEKOIA-IO/documentation/main/docs/assets/operation_center/ingestion_methods/sekoiaio_docker_concentrator/sekoiaio_docker_concentrator_autosetup.sh
    chmod +x sekoiaio_docker_concentrator_autosetup.sh
    ./sekoiaio_docker_concentrator_autosetup.sh
    rm sekoiaio_docker_concentrator_autosetup.sh
    ```

2. Edit the configuration files

    - `sekoiaio-concentrator/intakes.yaml` by replacing the `name`, `protocol`, `port` and `intake_key` for each intake you would like to collect 
    - `sekoiaio-concentrator/docker-compose.yml` by remplacing the value `"20516-20518:20516-20518"` by a relevant content according to the `sekoiaio-concentrator/intakes.yaml` previously edited

3. Start the docker

    Follow the process you can find on the section [Start the concentrator](https://docs.sekoia.io/xdr/features/collect/ingestion_methods/sekoiaio_docker_concentrator.md/#start-the-concentrator) of this page.
    ```bash
    sudo docker compose up -d
    sudo docker compose ps
    sudo docker compose logs -f
    ```

Enjoy your docker!
