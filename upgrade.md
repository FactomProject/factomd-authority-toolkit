# Security Update
The new Docker Swarm certificate and key is used for authenticating with the docker swarm.  This change will verify clients with the certificate, as well as encrypt communication with the Docker API using TLS. First, move the new keys to `/etc/docker` in order to use it for the docker api:

```

sudo wget https://raw.githubusercontent.com/FactomProject/factomd-authority-toolkit/master/tls/cert_exp_5-14-21.pem -O /etc/docker/factom-mainnet-cert_exp_5-14-21.pem
sudo wget https://raw.githubusercontent.com/FactomProject/factomd-authority-toolkit/master/tls/key_exp_5-14-21.pem -O /etc/docker/factom-mainnet-key_exp_5-14-21.pem
sudo wget https://raw.githubusercontent.com/FactomProject/factomd-authority-toolkit/master/tls/ca_exp_5-14-21.pem -O /etc/docker/factom-mainnet-ca_exp_5-14-21.pem
sudo chmod 644 /etc/docker/factom-mainnet-cert_exp_5-14-21.pem
sudo chmod 440 /etc/docker/factom-mainnet-key_exp_5-14-21.pem /etc/docker/factom-mainnet-ca_exp_5-14-21.pem
sudo chgrp docker /etc/docker/*.pem

```
## Choose one of the following options for configuring dockerd

### 1. Using `daemon.json` (recommended)

You can configure the docker daemon using a default config file, located at
`/etc/docker/daemon.json`. 

Example configuration:
```
{
  "tlsverify": true,
  "tlscert": "/etc/docker/factom-mainnet-cert_exp_5-14-21.pem",
  "tlskey": "/etc/docker/factom-mainnet-key_exp_5-14-21.pem",
  "tlscacert":"/etc/docker/factom-mainnet-ca_exp_5-14-21.pem",
  "hosts": ["tcp://0.0.0.0:2376", "unix:///var/run/docker.sock"]
}
```
 - *The line `tls` has been changed to `tlsverify`  and the `"tlscacert":"/etc/docker/factom-mainnet-ca_exp_5-14-21.pem",` line has been added*

As noted above, please make sure that you do not also specify any of these
options on the command line for `dockerd`. Please make sure to specify the
correct paths for `"tlscacert"`, `"tlscert"` and `"tlskey"`. If you are using `systemd` to run
the `docker.service` you will need an additional host in your host list:
`"fd://"`. See `systemd` below.

### 2. Options on the command line

For the same options as described above, you would use the following command line options:
```
dockerd -H=unix:///var/run/docker.sock -H=0.0.0.0:2376 --tlsverify --tlscacert=/etc/docker/factom-mainnet-ca_exp_5-14-21.pem --tlscert=/etc/docker/factom-mainnet-cert_exp_5-14-21.pem --tlskey=/etc/docker/factom-mainnet-key_exp_5-14-21.pem
```
## Choose one of the following 3 options for starting dockerd

### 1. On RedHat
Open (using `sudo`) `/etc/sysconfig/docker` in your favorite text editor.

Append `-H=unix:///var/run/docker.sock -H=0.0.0.0:2376 --tlsverify --tlscacert=/etc/docker/factom-mainnet-ca_exp_5-14-21.pem --tlscert=/etc/docker/factom-mainnet-cert_exp_5-14-21.pem --tlskey=/etc/docker/factom-mainnet-key_exp_5-14-21.pem` to the pre-existing OPTIONS

Then, `sudo service docker restart`.

### 2. Using systemd
Run `sudo systemctl edit docker.service`. This creates an override directory at `/etc/systemd/system/docker.service.d/` and an override file called `override.conf`. Alternatively, you can create this directory and file manually and you can give the file a more descriptive name so long as it ends with `.conf`.

Edit the override file to match this:

```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd
```

and make sure that you add `"fd://"` to the `"hosts"` array in `/etc/docker/daemon.json` if you are using it for your config.
If you are *not* using `/etc/docker/daemon.json` use the following for your service file override.

```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// -H unix:///var/run/docker.sock -H tcp://0.0.0.0:2376 --tlsverify --tlscacert  /etc/docker/factom-mainnet-ca_exp_5-14-21.pem --tlscert /etc/docker/factom-mainnet-cert_exp_5-14-21.pem --tlskey /etc/docker/factom-mainnet-key_exp_5-14-21.pem
```

Then reload the configuration and the docker.service

```
sudo systemctl daemon-reload
sudo systemctl restart docker.service
```

### 3. I don't want to use a process manager

You can manually start the docker daemon via:

```
sudo dockerd -H=unix:///var/run/docker.sock -H=0.0.0.0:2376 --tlsverify --tlscacert=/etc/docker/factom-mainnet-ca_exp_5-14-21.pem --tlscert=/etc/docker/factom-mainnet-cert_exp_5-14-21.pem --tlskey=/etc/docker/factom-mainnet-key_exp_5-14-21.pem
```
or just
```
sudo dockerd
```
if you are using the `/etc/docker/daemon.json` file for configuration.


