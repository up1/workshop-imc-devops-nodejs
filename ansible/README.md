# Deploy employees API with [Ansible](https://docs.ansible.com/projects/ansible/latest/getting_started/index.html)

This playbook deploys `somkiat/employees-api:3.0` to a remote Docker server and verifies that TCP port 3000 is listening.

## Prerequisites

On the Ansible control machine:

- Ansible is installed.
- `sshpass` is installed for SSH password authentication.
- SSH access to the target server is configured.

On the target server:

- Docker Engine is installed and running.
- The SSH user can use `sudo` to access Docker.
- Port 3000 is allowed through the firewall when the API must be reached from other machines.

## Configure

Run these commands from the `ansible` directory:

```shell
ansible-galaxy collection install -r requirements.yml
```

Configure the server address and SSH username in `inventory.ini`:

```ini
[servers]
app-server ansible_host=159.223.46.153 ansible_user=root
```

The SSH password is requested at runtime, so it is not stored in this file.

Verify SSH connectivity:

```shell
ansible servers -i inventory.ini --ask-pass -m ansible.builtin.ping
```

## Deploy

> **Warning:** The playbook force-removes every Docker container and every Docker volume on the target server before deployment. All data stored in Docker volumes will be permanently deleted.

Before starting the API, the playbook verifies that host port 3000 (or `api_host_port`) is no longer in use. Deployment fails if another process is still listening on that port.

```shell
ansible-playbook -i inventory.ini deploy.yml --ask-pass
```

Enter the SSH password when prompted. For a non-root SSH user whose `sudo` also requires a password, add `--ask-become-pass`.

The image and host port can be overridden:

```shell
ansible-playbook -i inventory.ini deploy.yml --ask-pass \
  -e api_image=somkiat/employees-api:3.0 \
  -e api_host_port=3000
```

Pass a PostgreSQL connection string when the employee endpoints need database access:

```shell
ansible-playbook -i inventory.ini deploy.yml --ask-pass \
  -e 'database_url=postgresql://postgres:postgres@DATABASE_HOST:5432/employees'
```

The database must already be reachable from the Docker server. This playbook deploys only the API container.

## Verify port 3000

The deployment fails automatically when port 3000 does not start listening within 30 seconds. Run the same check independently with:

```shell
ansible servers -i inventory.ini --ask-pass -b \
  -m ansible.builtin.wait_for \
  -a 'host=127.0.0.1 port=3000 state=started timeout=5'
```

Inspect the running container:

```shell
ansible servers -i inventory.ini --ask-pass -b \
  -m ansible.builtin.command \
  -a 'docker ps --filter name=employees-api'
```

From another machine, when the server firewall permits port 3000:

```shell
curl -i http://SERVER_IP:3000/
```

The root URL may return HTTP 404 because the API routes are under `/api/employees`; a response still confirms that the service is reachable.