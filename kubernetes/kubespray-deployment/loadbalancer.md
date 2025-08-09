```bash
sudo firewall-cmd --permanent --add-port={6443/tcp,2379-2380/tcp,10250/tcp,10251/tcp,10252/tcp}
sudo firewall-cmd --permanent --add-port=22/tcp  # SSH for Ansible
sudo firewall-cmd --reload
```

3. Verify open ports:

```bash
sudo firewall-cmd --list-ports
```
