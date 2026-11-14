# aetherion-evaluation

## Quick start

Set up Debian 13 hosts, form a Docker swarm, deploy the DeathStarBench socialNetwork app and load the social graph:

```sh
cd ansible

# 1. Add your hosts (first host in [swarm_managers] becomes the swarm leader)
cp inventory.example.ini inventory.ini
$EDITOR inventory.ini

# 2. Install the Ansible collection the swarm playbooks need
ansible-galaxy collection install -r requirements.yml

# 3. Install Docker, Python/aiohttp, luarocks, ... on all hosts
ansible-playbook -i inventory.ini setup.yml --ask-pass --ask-become-pass

# 4. Form the swarm, deploy the stack, register users and build the social graph
ansible-playbook -i inventory.ini socialnetwork.yml --ask-pass --ask-become-pass
```

The app is then available at `http://<any-node>:8080`, and the Jaeger UI at `http://<any-node>:16686`. `--ask-pass` prompts for the SSH password (needs `sshpass` on your machine) and `--ask-become-pass` for the `sudo` password. Leave out `--ask-pass` if you log in with an SSH key, and `--ask-become-pass` if `sudo` doesn't ask for a password.

To remove the stack again:

```sh
ansible-playbook -i inventory.ini socialnetwork-teardown.yml --ask-pass --ask-become-pass
```

The sections below explain each step and its options.

## Host setup (Ansible)

`ansible/setup.yml` provisions a **Debian 13 (trixie)** host with everything the evaluation needs:

- Docker Engine and Docker Compose (`docker compose`, plus a `docker-compose` shim)
- Python 3 with `asyncio` and `aiohttp`
- `libssl-dev`, `zlib1g-dev` (`libz-dev`)
- `luarocks` and `luasocket`

### Prerequisites

- On your machine: `ansible-core` 2.15 or newer
- On the target: Debian 13, SSH access, and a user with `sudo` rights

### Usage

1. Copy the example inventory and add your host(s). `ansible/inventory.ini` is git-ignored, so your hosts stay local:

   ```sh
   cp ansible/inventory.example.ini ansible/inventory.ini
   ```

   Then edit it, for example:

   ```ini
   [swarm_managers]
   eval-node-01 ansible_host=192.0.2.10

   [swarm_workers]
   eval-node-02 ansible_host=192.0.2.11
   eval-node-03 ansible_host=192.0.2.12

   [eval:children]
   swarm_managers
   swarm_workers

   [eval:vars]
   ansible_user=debian
   ```

   `setup.yml` runs on every host. The swarm groups are used by the
   [socialNetwork playbooks](#socialnetwork-on-docker-swarm-ansible).

2. Run the playbook:

   ```sh
   cd ansible
   ansible-playbook -i inventory.ini setup.yml --ask-pass --ask-become-pass
   ```

   Leave out `--ask-pass` if you log in with an SSH key, and `--ask-become-pass` if `sudo` doesn't need a password on the target.

3. Log out and back in on the target, so the `docker` group membership takes effect.

The last task prints the installed versions of Docker, Compose, Python/aiohttp, luarocks and luasocket, so you can check that everything was installed.

### Options

| Variable       | Default                  | Description                                                                 |
|----------------|--------------------------|-----------------------------------------------------------------------------|
| `docker_users` | the user who ran `sudo`  | Existing users to add to the `docker` group (users that don't exist are skipped) |
| `lua_version`  | `5.1`                    | Lua version luarocks builds for (`5.1`, `5.2` or `5.3`)                     |

Example:

```sh
ansible-playbook -i inventory.ini setup.yml --ask-pass --ask-become-pass \
  -e '{"docker_users": ["alice", "bob"]}' -e lua_version=5.3
```

To run against a single host without an inventory file:

```sh
ansible-playbook -i 192.0.2.10, -u debian --ask-pass --ask-become-pass setup.yml
```

## socialNetwork on Docker swarm (Ansible)

`ansible/socialnetwork.yml` deploys the [DeathStarBench socialNetwork](https://github.com/delimitrou/DeathStarBench/blob/master/socialNetwork/README.md) app on a Docker swarm, then registers users and builds the social graph. It does the following:

1. **Prepares every node** (tag `swarm`, `deploy`): installs `git` and `python3-docker`, and clones DeathStarBench to `/opt/DeathStarBench`. Swarm bind-mounts the app's config files, so the repo must be at the same path on every node.
2. **Forms the swarm** (tag `swarm`): the first host in `[swarm_managers]` runs `swarm init`. Other managers join as managers and `[swarm_workers]` join as workers. Nodes already in the swarm are left alone.
3. **Deploys the stack** (tag `deploy`): runs `docker stack deploy --compose-file docker-compose-swarm.yml socialnetwork` on the leader. It then waits until every service has all its replicas running and the nginx frontend answers on port 8080.
4. **Registers users and constructs the social graph** (tag `init`): runs `scripts/init_social_graph.py` on the leader against the frontend.

### Prerequisites

- `setup.yml` has been run on all nodes (Docker, Python + aiohttp).
- The `community.docker` collection is installed on your machine:

  ```sh
  ansible-galaxy collection install -r ansible/requirements.yml
  ```

- The nodes can reach each other on the swarm ports: `2377/tcp`, `7946/tcp+udp` and `4789/udp`.
- `amd64` nodes. The upstream DeathStarBench images target `amd64`, so check image support before using other architectures.

### Usage

```sh
cd ansible

# Form swarm, deploy stack, load the default (small) graph
ansible-playbook -i inventory.ini socialnetwork.yml --ask-pass --ask-become-pass

# Use a larger graph
ansible-playbook -i inventory.ini socialnetwork.yml -e dsb_graph=ego-twitter --ask-pass --ask-become-pass

# Only (re)deploy the stack, or only load the graph
ansible-playbook -i inventory.ini socialnetwork.yml --tags deploy --ask-pass --ask-become-pass
ansible-playbook -i inventory.ini socialnetwork.yml --tags init --ask-pass --ask-become-pass
```

Once deployed, the frontends are reachable on any swarm node:

| Port    | Service             |
|---------|---------------------|
| `8080`  | nginx web frontend  |
| `8081`  | media frontend      |
| `16686` | Jaeger UI           |

#### Re-running the graph initialisation

`init_social_graph.py` is not idempotent (it registers every user again), so the init step runs only once per stack. After a successful run it writes a marker file, `/var/lib/deathstarbench/socialnetwork.social-graph`, on the leader. The marker is removed automatically when the stack is freshly created, because the databases are empty again. To force a re-run anyway:

```sh
ansible-playbook -i inventory.ini socialnetwork.yml --tags init -e dsb_init_force=true --ask-pass --ask-become-pass
```

To switch to a different graph, tear the stack down first (see below). Loading a second graph into the same databases mixes the user IDs.

### Options

All defaults are in `ansible/group_vars/all/socialnetwork.yml`.

| Variable                 | Default                    | Description                                                                 |
|--------------------------|----------------------------|-----------------------------------------------------------------------------|
| `dsb_version`            | `master`                   | DeathStarBench branch, tag or commit to check out                           |
| `dsb_dir`                | `/opt/DeathStarBench`      | Checkout path (same on every node)                                          |
| `dsb_stack_name`         | `socialnetwork`            | Name of the swarm stack                                                     |
| `swarm_advertise_addr`   | host's default IPv4        | Address each node advertises to the swarm                                  |
| `dsb_converge_retries`   | `90`                       | How many 10 s checks to wait for all services (the first deploy pulls all images) |
| `dsb_graph`              | `socfb-Reed98`             | `socfb-Reed98` (small), `ego-twitter` (medium), `soc-twitter-follows-mun` (large) |
| `dsb_init_limit`         | `200`                      | Number of concurrent requests during initialisation                         |
| `dsb_init_compose_posts` | `false`                    | Also compose posts for each user (`--compose`)                              |
| `dsb_init_force`         | `false`                    | Re-run initialisation even if the marker exists                             |

TLS is not enabled. To use it, edit the config files listed in the DeathStarBench README (`config/mongod.conf`, `config/redis.conf`, `config/service-config.json`, `nginx-web-server/conf/nginx.conf`) on every node, then redeploy.

### Reset

To get a fresh app between performance tests, remove the stack, deploy it again and reload the social graph in one run. The swarm stays up, and the images are already on the nodes:

```sh
ansible-playbook -i inventory.ini socialnetwork-reset.yml --ask-pass --ask-become-pass
```

`-e dsb_graph=...` and the other `socialnetwork.yml` options work here too.

### Teardown

```sh
# Remove the stack (all data is lost); the swarm stays up
ansible-playbook -i inventory.ini socialnetwork-teardown.yml --ask-pass --ask-become-pass

# Also dissolve the swarm
ansible-playbook -i inventory.ini socialnetwork-teardown.yml -e swarm_leave=true --ask-pass --ask-become-pass
```
