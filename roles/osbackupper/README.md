# OSBackupper

## Usage

An exemplary palybook:

```yml
- hosts: backup-servers
  become: true
  vars_files:
    - vars/vault.yml   # encrypted with ansible-vault; must define openstack_auth
  roles:
    - openstackbackupper
```

## Variables

In your vault file:

```yml
openstack_auth:
  OS_AUTH_URL: "https://openstack.example.com:5000/v3"
  # Prefer application credential:
  OS_APPLICATION_CREDENTIAL_ID: "your-app-cred-id"
  OS_APPLICATION_CREDENTIAL_SECRET: "your-app-cred-secret"
  # OR username/password (less recommended)
  # OS_USERNAME: "backupuser"
  # OS_PASSWORD: "secret"
  OS_PROJECT_ID: "your-project-id"
  OS_USER_DOMAIN_NAME: "Default"
  OS_PROJECT_DOMAIN_NAME: "Default"
  OS_REGION_NAME: "RegionOne"
```
