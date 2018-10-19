# Factomd Control Plane

This repository is responsible for maintaining the control plane for factomd M3.

It includes 3 containers:
  1. Factomd
  > Runs the factom node
  2. SSH
  > Permits ssh access only to _this specific container_. Mounts the factomd database volume for debugging purposes.
  3. Filebeat
  > Reports stdout/stderr of all docker containers to our elasticsearch instance

# Install docker

Please follow the instructions [here](https://docs.docker.com/install/linux/docker-ce/ubuntu/) to install docker-ce to your machine. If you run Ubuntu 18.04 you can use the docker.io package `sudo apt-get install docker.io` as it's recent enough to support swarm and iptables without modification.

Then, run `usermod -aG docker $USER` and make sure you do logout/login, as otherwise your current terminal session will not be updated.


# Configure Docker

In order to join the swarm, first ensure that your firewall rules allow access on the following ports. All swarm communications occur over TLS using a self-signed TLS certificate. Due to the way iptables and docker work you cannot use the `INPUT` chain to block access to apps running in a docker container as it's not a local destination but a `FORWARD` destination. By default when you map a port into a docker container it opens up to `any` host. To restrict access we need to add our rules in the `DOCKER-USER` chain [reference](https://docs.docker.com/network/iptables/).

- TCP port `2376` _only to_ `52.48.130.243 & 18.203.51.247` for secure Docker engine communication. This port is required for Docker Machine to work. Docker Machine is used to orchestrate Docker hosts. As this is a local service we use the `INPUT` chain.

In addition,  the following ports must be opened for factomd to function which we add to the `DOCKER-USER` chain:
- `2222` to `52.48.130.243 & 18.203.51.247`, which is the SSH port used by the `ssh` container
- `8088` to `52.48.130.243 & 18.203.51.247`, the factomd API port
- `8090` to `52.48.130.243 & 18.203.51.247`, the factomd Control panel
- `8108` to the world, the factomd mainnet port

An example using `iptables`:
```
sudo iptables -A INPUT ! -s 52.48.130.243/32 -p tcp -m tcp --dport 2376 -m conntrack --ctstate NEW,ESTABLISHED -j REJECT --reject-with icmp-port-unreachable
sudo iptables -A DOCKER-USER ! -s 52.48.130.243/32  -i <external if> -p tcp -m tcp --dport 8090 -j REJECT --reject-with icmp-port-unreachable
sudo iptables -A DOCKER-USER ! -s 52.48.130.243/32  -i <external if> -p tcp -m tcp --dport 2222 -j REJECT --reject-with icmp-port-unreachable
sudo iptables -A DOCKER-USER ! -s 52.48.130.243/32  -i <external if> -p tcp -m tcp --dport 8088 -j REJECT --reject-with icmp-port-unreachable
```
Repeat these instructions with the IP 18.203.51.247

(Replace `<external if>` with the name of the interface you use to connect to the internet eg. eth0 or ens0. To see interfaces use `ip addr list`)

Don't forget to [save](https://www.digitalocean.com/community/tutorials/iptables-essentials-common-firewall-rules-and-commands#saving-rules) the rules!

# Store Docker Swarm certificate and key
Make sure you store the Docker Swarm mainnet key and certificate on your system. The files can be found at [here](https://github.com/FactomProject/factomd-authority-toolkit/tree/master/tls)
You can store these files in the directory /etc/docker for instance:
```
sudo mkdir -p /etc/docker
sudo wget https://raw.githubusercontent.com/FactomProject/factomd-authority-toolkit/master/tls/cert.pem -O /etc/docker/factom-mainnet-cert.pem
sudo wget https://raw.githubusercontent.com/FactomProject/factomd-authority-toolkit/master/tls/key.pem -O /etc/docker/factom-mainnet-key.pem
sudo chmod 644 /etc/docker/factom-mainnet-cert.pem
sudo chmod 440 /etc/docker/factom-mainnet-key.pem
sudo chgrp docker /etc/docker/*.pem
```

Please note that in the rest of this file we assume you stored the files using the /etc/docker location and with above names. If not please adjust the commands below involving the certificate and keys.

# Configure and Run the Docker Engine

There are a number of ways to run `dockerd` and two effectively mutually
exclusive ways to configure `dockerd`. The ways to run `dockerd` are discussed
below, but it is also important to understand the two ways that it can be
configured.

## Choose one of the following options for configuring `dockerd`
You can either use the `/etc/docker/daemon.json` file to specify `dockerd`
options, or you can specify options on the command line. Note that while these
methods can be used together, *if the same option is specified in both
locations, `dockerd` will fail to start even if the options agree*. For this
reason it is best to either specify all options on the command line or all
options in `/etc/docker/daemon.json`.

### 1. Using `daemon.json` (recommended)

You can configure the docker daemon using a default config file, located at
`/etc/docker/daemon.json`. Create this file if it does not exist.

Example configuration:
```
{
  "tls": true,
  "tlscert": "/etc/docker/factom-mainnet-cert.pem",
  "tlskey": "/etc/docker/factom-mainnet-key.pem",
  "hosts": ["tcp://0.0.0.0:2376", "unix:///var/run/docker.sock"]
}
```
As noted above, please make sure that you do not also specify any of these
options on the command line for `dockerd`. Please make sure to specify the
correct paths for `"tlscert"` and `"tlskey"`. If you are using `systemd` to run
the `docker.service` you will need an additional host in your host list:
`"fd://"`. See `systemd` below.

### 2. Options on the command line

For the same options as described above, you would use the following command
line options:

```
dockerd -H=unix:///var/run/docker.sock -H=0.0.0.0:2376 --tls --tlscert=/etc/docker/factom-mainnet-cert.pem --tlskey=/etc/docker/factom-mainnet-key.pem
```

## Choose one of the following 3 options for starting dockerd
Remeber that if you specify an option on the command line, you can't have the
same option in your `/etc/docker/daemon.json` file.
### 1. On RedHat

Open (using `sudo`) `/etc/sysconfig/docker` in your favorite text editor.

Append `-H=unix:///var/run/docker.sock -H=0.0.0.0:2376 --tls --tlscert=/etc/docker/factom-mainnet-cert.pem --tlskey=/etc/docker/factom-mainnet-key.pem` to the pre-existing OPTIONS

Then, `sudo service docker restart`.

### 2. Using `systemd`

Run `sudo systemctl edit docker.service`. This creates an override directory at
`/etc/systemd/system/docker.service.d/` and an override file called
`override.conf`. Alternatively, you can create this directory and file manually
and you can give the file a more descriptive name so long as it ends with
`.conf`.

Edit the override file to match this:
```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd
```
and make sure that you add `"fd://"` to the `"hosts"` array in
`/etc/docker/daemon.json` if you are using it for your config.

If you are *not* using `/etc/docker/daemon.json` use the following for your
service file override.
```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// -H unix:///var/run/docker.sock -H tcp://0.0.0.0:2376 --tls --tlscert /etc/docker/factom-mainnet-cert.pem --tlskey /etc/docker/factom-mainnet-key.pem
```
Then reload the configuration and the `docker.service`
```
sudo systemctl daemon-reload
sudo systemctl restart docker.service
```

### 3. I don't want to use a process manager

You can manually start the docker daemon via:

```
sudo dockerd -H=unix:///var/run/docker.sock -H=0.0.0.0:2376 --tlscert=/etc/docker/factom-mainnet-cert.pem --tlskey=/etc/docker/factom-mainnet-key.pem
```
or just
```
sudo dockerd
```
if you are using the `/etc/docker/daemon.json` file for configuration.

## Troubleshooting

If `dockerd` fails to start review the error output carefully. It generally
tells you exactly what the problem is.

If you are using systemd and the service fails to start, finding the relevant
logs can be a challenge since the service is configured to just keep restarting
which can bury the logs.

In this case, stop the service: `sudo systemctl stop docker`.

Then manually start `dockerd`: `sudo dockerd`

You will then be able to see the `dockerd` output which should point you at the
problem. Fix those and then try starting the service with systemd again.

# Create the Factomd volumes
Factomd relies on two volumes,`factom_database` and `factom_keys`. Please create these before joining the swarm.

1. `docker volume create factom_database`
2. `docker volume create factom_keys`

These volumes will be used to store information by the `factomd` container.

For running a mainnet authority node we strongly recommend syncing the database from scratch. If you do already have a recently synced mainnet node and would like to avoid resyncing, run:

`sudo cp -r <path to your database> /var/lib/docker/volumes/factom_database/_data`.

The directory in `_data` after the copy should be `main-database`, as the volume is mounted at `$HOME/.factom/m2`.

In addition, please place your `factomd.conf` file in `/var/lib/docker/volumes/factom_keys/_data`. An example factomd.conf can be found [here](https://raw.githubusercontent.com/FactomProject/factomd/master/factomd.conf)

# Join the Docker Swarm

Finally, to join the swarm:
```
docker swarm join --token SWMTKN-1-5ct5plmbn1ombbjqp8ql8hq93jkof6246suzast5n1gfwa083b-1ui6w6fupe45tizz0tv6syzrs 52.48.130.243:2377

```

As a reminder, joining as a worker means you have no ability to control containers on another node.

Once you have joined the network, you will be issued a control panel login by a Factom employee after messaging one of the Factom engineers on discord. You should private message the following for **each** node:
- NodeID (`docker info`)
- IP Address
- Docker engine listening port (`2376`)

**Only accept logins at federation.factomd.com. Any other login endpoints are fraudulent and not to be trusted.**

# Starting Factomd Container from the Docker CLI

Please note: There is a version for the Factom software in the next command. Make sure you run the correct and latest announced version from the Discord #operators-announcement channel

Run this command _exactly_: `docker run -d --name "factomd" -v "factom_database:/root/.factom/m2" -v "factom_keys:/root/.factom/private" -p "8088:8088" -p "8090:8090" -p "8108:8108" -l "name=factomd" factominc/factomd:v5.2.0-alpine -startdelay=600 -faulttimeout=120 -config=/root/.factom/private/factomd.conf
`
After this your node will be started. You can check for the existence of a factom container using the command `docker ps`
Please wait for your node to be fully synced by checking the control panel node sync statuses to be 100% before performing any next steps. Please also regard the initial wait period of 20 minutes before doing anything with your node.  Also note that the factomd:v5.X.X-alpine version may have changed since the time of writing.

### NOTE: The Swarm cluster is still experimental, so please pardon our dust! If you have an issues, please contact ian at factom dot com.
