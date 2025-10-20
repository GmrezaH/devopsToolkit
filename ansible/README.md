Install prerequisite packages on ansible server

```bash
sudo dnf install -y python3.12 python3.12-pip
```

Create a Python virtual environment and Install required pypi packages

```bash
python3.12 -m venv venv
source venv/bin/activate
cd ansible
python3.12 -m pip install ansible
python3.12 -m pip install -U  -r requirements.txt
```

Install required ansible-galaxy collections:

```bash
# on online vm
ansible-galaxy collection download community.docker -p ./collections

# on offline vm
ansible-galaxy collection install collections/community-docker-4.7.0.tar.gz
ansible-galaxy collection list | grep docker
```

Install prerequisite packages on destination servers

```bash
ansible all -i inventory/hosts.yml  -a "dnf install -y python3.12 python3.12-pip python3-libselinux"
```

Run playbook in check mode (without making any changes to target hosts) before the actual run

```bash
ansible-playbook playbook/hardening.yml --check
```
