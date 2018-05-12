## Being a swarm manager

### Onboarding a node

1. Once a node has joined the swarm, label their nodes with `name=<desired name>`, `ip=<public ip>`, and `engine_port=<2376 unless otherwise specified>`. This is done via `docker node update --label-add key=value <node id>` from the manager node.
- `docker node ls` to confirm joining.
- `docker node update --label-add name=team-001 --label-add ip=1.2.3.4 --label-add engine_port=2376 1qeroijqr45134614561345` 
- `docker node inspect 1qeroijqr45134614561345 | less`

2. Login to the [manager control panel](https://federation.factomd.com), and click "endpoints." Then "add an endpoint." Enter their public ip & engine port in the endpoint URL `1.2.3.4:2376`, and name the endpoint the same name as used above. Then, click "TLS" and "TLS with client verification." Upload the `cert.pem` and `key.pem` files in this repository. Then click add.
3. Click "Users." Create a username and password for that node.
4. Click "Endpoints" again, and "manage access." Then add the new user to that endpoint.
5. Distribute the username and password to the node operator.

### Restarting the network

This is a simple process. From the manager node, run `python factomd_control/utils/restart_all.py`.
