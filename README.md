# Factomd Control Plane

This repository is responsible for maintaining the control plane for factomd M3.

It includes 4 containers:
  1. FactomD
  > Runs the factom node
  2. SSH
  > Permits ssh access only to _this specific container_. Mounts the factomd database volume for debugging purposes.
  3. Filebeat
  > Reports stdout/stderr of all docker containers to our elasticsearch instance
  4. Metricbeat
  > Reports hardware metrics of all docker containers to our elasticsearch instance.

## Install docker

Please follow the instructions [here](https://docs.docker.com/install/linux/docker-ce/ubuntu/) to install docker-ce to your machine.

Then, run `usermod -aG docker $USER` and logout/login.


## How to join the Swarm

In order to join the swarm, first ensure that your firewall rules allow access on the following ports. All swarm communications occur over a self-signed TLS certificate.

- TCP port `2376` _only to_ `52.48.130.243` for secure Docker engine communication. This port is required for Docker Machine to work. Docker Machine is used to orchestrate Docker hosts.

An example using `iptables`:
- `sudo iptables -A INPUT -p tcp -s 52.48.130.243 --dport 2376 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT`

- `sudo service iptables save`


In addition,  the following ports must be opened for factomd to function:
- `2222`, which is the SSH port used by the `ssh` container
- `8108`, the factomd mainnet port
- `8088`, the factomd API port

Once these ports are open, create two volumes on your host machine:

1. `docker volume create factom_database`
2. `docker volume create factom_keys`

These volumes will be used to store information by the `factomd` container.

If you already have a synced node and would like to avoid resyncing, run:

`sudo cp -r <path to your database> /var/lib/docker/volumes/factom_database/_data`.

In addition, please place your `factomd.conf` file in `/var/lib/docker/volumes/factom_keys/_data`.

## Exposing the Docker Engine

### Using `systemd`

Open (using `sudo`) `/etc/sysconfig/docker` in your favorite text editor.

Append `-H=unix:///var/run/docker.sock -H=0.0.0.0:2376 --tlscert=<path to cert.pem> --tlskey=<path to key.pem>` to the pre-existing OPTIONS

Then, `sudo service docker restart`.

### Using `systemctl`

Run `sudo systemctl edit docker.service`

Edit the file to match this:

```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// -H unix:///var/run/docker.sock -H tcp://0.0.0.0:2376 --tlscert <path to cert.pem> --tlskey <path to key.pem>
```

Then reload the configuration:
`sudo systemctl daemon-reload`

and restart docker:

`sudo systemctl restart docker.service`

### I don't want to use a process manager

You can manually start the docker daemon via:
`sudo dockerd -H=unix:///var/run/docker.sock -H=0.0.0.0:2376 --tlscert=<path to cert.pem> --tlskey=<path to key.pem>`.

Finally, to join the swarm:
```
docker swarm join --token SWMTKN-1-1wnjsb3jyt573yg9a765rua50ytkgcbkuur55m7y2haoc4ze2q-engcpbkxm53gqfociyiz71naa 54.171.68.124:2377
```

As a reminder, joining as a worker means you have no ability to control containers on another node.

Once you have joined the network, you will be issued a control panel login by a Factom employee.

**Only accept logins at federation.factomd.com. Any other login endpoints are fraudulent and not to be trusted.**

## Starting FactomD Container

There are two means of launching your `factomd` instance:

### From the Docker CLI

Run this command _exactly_: `docker run -d -v "factom_database:/root/.factom/m2" -v "factom_keys:/root/.factom/private" -p "8088:8088" -p "8090:8090" -p "8108:8108" -l "name=factomd" factominc/factomd:v5.0.0-alpine`

### From the Portainer UI

Once you have logged into the control panel, please ensure your node is selected in the top left dropdown menu.

Then, click `containers > add container`.

**Instructions marked with :heavy_exclamation_mark: _must_ be followed exactly or you risk being booted from the authority set.**

1. Name your container as you see fit

:heavy_exclamation_mark: 2. Enter the image name `factominc/factomd:v5.0.0-alpine`

:heavy_exclamation_mark: 3. Mark additional ports `8088:8088`, `8009`:`8009`, `8090:8090`.

:heavy_exclamation_mark: 4. Do _not_ modify access control.

5. Either leave the bootup command blank, or create or your own. Be careful!

:heavy_exclamation_mark: 6. Click "volumes", and map `/root/.factom/m2` to `factom_database`, and `/root/.factom/private` to `factom_keys`.

:heavy_exclamation_mark: 7. Click "labels" and add a label `name:name` = `value:factomd`

:heavy_exclamation_mark: 8. Click "deploy the container"

9. You are done!


### NOTE: The Swarm cluster is still experimental, so please pardon our dust! If you have an issues, please contact ian at factom dot com.

## Being a swarm manager

### Onboarding a node

1. Give the target node the `cert.pem` and `key.pem` files in a secure manner, and have them place it in a reasonable location on their machine
2.

The network can be restarted with `python restart_all.py`, executed from the manager node.
