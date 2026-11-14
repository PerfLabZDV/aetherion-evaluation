# aetherion-evaluation

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
   [eval]
   eval-node-01 ansible_host=192.0.2.10 ansible_user=debian
   ```

2. Run the playbook:

   ```sh
   cd ansible
   ansible-playbook -i inventory.ini setup.yml
   ```

   Add `-K` (`--ask-become-pass`) if `sudo` needs a password on the target.

3. Log out and back in on the target, so the `docker` group membership takes effect.

The last task prints the installed versions of Docker, Compose, Python/aiohttp, luarocks and luasocket, so you can check that everything was installed.

### Options

| Variable       | Default                  | Description                                                                 |
|----------------|--------------------------|-----------------------------------------------------------------------------|
| `docker_users` | the user who ran `sudo`  | Existing users to add to the `docker` group (users that don't exist are skipped) |
| `lua_version`  | `5.1`                    | Lua version luarocks builds for (`5.1`, `5.2` or `5.3`)                     |

Example:

```sh
ansible-playbook -i inventory.ini setup.yml \
  -e '{"docker_users": ["alice", "bob"]}' -e lua_version=5.3
```

To run against a single host without an inventory file:

```sh
ansible-playbook -i 192.0.2.10, -u debian -K setup.yml
```
