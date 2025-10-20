add repo.testbed.moh to both ansible and target vm /etc/hosts
manually create cleanup policies

create vault variables:

```bash
cat <<EOF > roles/gitlab/defaults/main/vault.yml
# Nexus Credentials
nexus_username: admin
nexus_password: 3903f1e1c17417f443f79d758091e046bldifug79f5d7270c8

# Nexus Users
other_user_password: fe1088eee7slkv96f390d19d767aa98e8b6c3c0e9fb1

EOF

ansible-vault encrypt roles/gitlab/defaults/main/vault.yml
```

