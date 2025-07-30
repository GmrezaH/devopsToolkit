# SSH config

The SSH configuration file (`~/.ssh/config`) allows you to define connection settings for SSH clients, making it easier to connect to remote servers without repeatedly specifying options like hostnames, usernames, ports, or keys.

## File location

If the file doesn't exist, create it with appropriate permissions:

```bash
touch ~/.ssh/config
chmod 600 ~/.ssh/config
```

## Basic syntax

Each entry in the `config` file defines settings for a specific host or group of hosts. The basic structure is:

```
Host <alias>
    HostName <remote_host>
    User <username>
    Port <port_number>
    IdentityFile <path_to_private_key>
    StrictHostKeyChecking yes
```

> [!NOTE]
> Note that keywords are case-insensitive and arguments are case-sensitive

## Sample configurations

1. ProxyJump

   ProxyJump allows SSH to connect to a target server through a bastion host (jump server)

   ```
   Host bastion
       HostName bastion.example.com
       User bastionuser
       Port 22
       IdentityFile ~/.ssh/bastion_key

   Host internal
       HostName 10.0.0.5
       User internaluser
       Port 22
       IdentityFile ~/.ssh/internal_key
       ProxyJump bastion
   ```

2. ProxyCommand

   When the bastion host requires a different user or port, or for older SSH versions that don’t support `ProxyJump`, use ProxyCommand with nc (netcat) or ssh

   ```
   Host internal
       HostName 10.0.0.5
       User internaluser
       Port 22
       IdentityFile ~/.ssh/internal_key
       ProxyCommand ssh -W %h:%p -q {BASTION_USER}:{BASTION_HOST}
   ```

3. LocalForward for Port Forwarding

   LocalForward binds a local port to a remote server’s port, allowing you to access services on the remote server as if they were running locally.

   ```
   Host myserver
       HostName example.com
       User johndoe
       Port 22
       IdentityFile ~/.ssh/id_rsa
       LocalForward 8080 localhost:80
   ```

4. Using wildcards

   You can use wildcards to apply settings to multiple hosts

   ```
   Host *.example.com
       User shareduser
       IdentityFile ~/.ssh/example_key
   ```

## Resources

- [ssh_config man page](https://linux.die.net/man/5/ssh_config)

- [nerderati.com](https://nerderati.com/2011-03-17-simplify-your-life-with-an-ssh-config-file/)
