# Deploy Quay HA with Postgres/Patroni cluster using etcd and MinIO

## Overview

Currently used Quay version: 3.3.1

This repository stores an example of Ansible roles which will help to deploy a High Available Red Hat Quay Enterprise Registry using the following components:

* Patroni to spin up a PostGres cluster, using etcd as DCS
* MinIO as quick and easy S3 storage backend
* Containers for Quay, Clair, Redis ...

The architecture in this example assumes that there are 4 nodes for Quay containers, while on 3 of these nodes etcd are running and 3 extra nodes for MinIO storage. The assumption is that Quay standalone (outside of OpenShift) is used. While Patroni, Postgres and MiniIO do NOT run inside containers, all Quay elements do. This way it is easy to scale Quay itself.

Since we are talking about HA some kind of loadbalancer must be used to balance the services. The automation of a LB is out of scope for these roles, but an example HAProxy configuration has been added to the repository.

## Nodes

The following nodes are assumed in this example (Change defaults and vars accordingly if you want to change something):

psql01 (192.168.100.4): Patroni, etcd, Quay, Clair, Redis
psql02 (192.168.100.5): Patroni, etcd, Quay, Clair, Redis
psql03 (192.168.100.6): Patroni, etcd, Quay, Clair, Redis
psql04 (192.168.100.7): Patroni, etcd, Quay, Clair, Redis
minio01 (192.168.101.4): Minio
minio02 (192.168.101.5): Minio
minio03 (192.168.101.6): Minio

The following ports must be load balanced (all TCP):

* Quay: 443 (80 optional)
* Quay-Config: 8443 --> runs on a single node and only when you need to configure something
* Redis: 6379
* Clair: 6060
* Postgres: 5432
* Minio: 9000

**NOTE**: Redis a actually not considered as a critical service, since it is only responsible for builder logs and the Quay Tutorial.

## Structure

### Inventory

The example inventory files looks like the following:

```ini
[psql]
psql01 ansible_host=192.168.100.4
psql02 ansible_host=192.168.100.5
psql03 ansible_host=192.168.100.6
psql04 ansible_host=192.168.100.7

[etcd_master]
psql01
psql02
psql03

[etcd]
psql01
psql02
psql03

[quay_config]
psql01

[minio]
minio01 ansible_host=192.168.101.4
minio02 ansible_host=192.168.101.5
minio03 ansible_host=192.168.101.6
```

The Quay-Config container will only be started on the node __psql01__. 

### Variables - Defined as group_vars

The following group_vars are defined for all nodes.

```yaml
base_domain: ispworld.at # defines the base domain for quay, clair and minio
use_local_files: true # should be true if no access to Epel or other repositories is possible. In this case Minio binary and required RPMs are copied from the Ansible roles

etc_hosts_add_extra_entries: True # Only required if DNS resolving is not configured
etc_hosts_extra_entries:
  - ip: 192.168.101.4
    hostnames: 'minio01.ispworld.at minio01'
  - ip: 192.168.101.5
    hostnames: 'minio02.ispworld.at minio02'
  - ip: 192.168.101.6
    hostnames: 'minio03.ispworld.at minio03'

######################
#     Patroni DB     #
######################
postgres_replication_username: "replication" # Username for the replication user

######################
#       MinIO        #
######################
minio_access_key: minioadmin # Username for Minio
minio_server_cluster_nodes: # list of minio members. Each has 4 directories export1 to export4
  - "https://minio01.ispworld.at:9000/export{1...4}"
  - "https://minio02.ispworld.at:9000/export{1...4}"
  - "https://minio03.ispworld.at:9000/export{1...4}"
minio_tls: true
minio_hostname: "minio.{{ base_domain }}" 
minio_ssl_crt: "{{ playbook_dir }}/files/minio-certs/fullchain.pem" # Chain certificate for Minio
minio_ssl_key: "{{ playbook_dir }}/files/minio-certs/privkey.pem" # Key for the certificate

minio_server_datadirs: 
  - /export1
  - /export2
  - /export3
  - /export4
minio_server_make_datadirs: true
minio_initial_bucket: "quay" # Initial bucket must be created before Quay is using minio

######################
#   Quay Deployment  #
######################
quay_hostname: quay.{{ base_domain }}
quay_version: v3.3.1
clair_version: v3.3.1
quay_db_host: 192.168.100.1 # cluster VIP

# Define SSL certs for Quay
quay_ssl_cert: "{{ playbook_dir }}/files/ssl.cert"
quay_ssl_key: "{{ playbook_dir }}/files/ssl.key"

# Define ca bundle if any. If not, leave empty
#quay_ca_bundle: "{{ playbook_dir }}files/ca.crt"

######################
#       Redis        #
######################
quay_redis_host: "192.168.100.1" # IP/Hostname to reach Redis

######################
#        Clair       #
######################
clair_url: http://192.168.100.1:6060 # IP/Hostname to reach Redis
clair_db_host: 192.168.100.1
clair_ssl_cert: "{{ playbook_dir }}/files/ssl.cert"
clair_ssl_key: "{{ playbook_dir }}/files/ssl.key"

######################
# Patroni Deployment #
######################
db_network: "192.168.100.0/24"
patroni_etcd_hosts: 192.168.100.4:2379,192.168.100.5:2379,192.168.100.6:2379 # list of etcd member, create certificate with IP SAN first

######################
#   etcd settings    #
######################
etcd_cluster_name: etcd.cluster
etcd_initial_cluster_token: 32255c24-1706-5eb7-8e14-59ccc89e20b5 # cluster token which should be created
etcd_master_group_name: etcd_master
etcd_secure: True # use TLS or not, if not the following settings can be ignored

# Certificate for the server to server communication
etcd_ssl:
  # Certificate file
  - src: "{{ playbook_dir }}/files/etcd-certs/etcd-ca/certs/etcd0.example.com.crt"
    mode: '0600'
    dst: "etcd0.example.com.crt"
  # Certificate KEY file
  - src: "{{ playbook_dir }}/files/etcd-certs/etcd-ca/private/etcd0.example.com.key"
    mode: '0400'
    dst: "etcd0.example.com.key"
  # CA file
  - src: "{{ playbook_dir }}/files/etcd-certs/etcd-ca/certs/ca.crt"
    dst: "ca.crt"
    mode: '0600'

# Certificate for client to server communication.
etcd_client_cert:
  # Certificate file
  - src: "{{ playbook_dir }}/files/etcd-certs/etcd-ca/certs/etcd-client.crt"
    mode: '0600'
    dst: "etcd-client.crt"
  # Certificate KEY file
  - src: "{{ playbook_dir }}/files/etcd-certs/etcd-ca/private/etcd-client.key"
    mode: '0400'
    dst: "etcd-client.key"
  # CA file
  - src: "{{ playbook_dir }}/files/etcd-certs/etcd-ca/certs/ca.crt"
    dst: "ca.crt"
    mode: '0600'
```

### Sensitive Variables - Defined in a vault

The following example vault has been created and should be checked before deployment:

```yaml
 # Quay.io Registry
 quayio_pass: <Password to authenticate at Quay.io>

 # Quay Database password
 quay_db_pass: <database password for Quay>

 # Quay Password for config UI
 quay_config_pass: <Password for Quay Config>

 # Quay Superuser
 quay_superuser: <Quay Superusername>
 quay_superuser_password: <Quay Superuser Password> # MUST be at least 8 chars long
 quay_superuser_email: <Quay Superuser Email>

 # Clair Database Password
 clair_db_pass: <Clair database password>

 # Minio KEY
 minio_secret_key: "<minio password>" # must be 8 chars minimum

 # Patroni DB
 postgres_superuser_password: "<superuser password>"
 postgres_replication_password: "<password for replication user>"
```

Other settings in the different roles, might be modified if required.

## Installation

The deployment should be done in the following order. It is also possible to use the __all__ playbook which would call eery role after each other. However, to get more control about what is happening the step-ba-step installation is explained here:

### Deploy MinIO

1. Check the appropriate settings in the group_vars and vault. Be sure to change the password, which must be 8 characters at least.
2. Execute:

   ```bash
   ansible-playbook -i inventory/inventory playbooks/deploy_minio.yaml --ask-vault-pass
   ```

### Deploy ETCD

1. Check the appropriate settings in the group_vars and vault.
2. Create self-signed certificates for the ETCD cluster. Following: https://github.com/kelseyhightower/etcd-production-setup
3. Execute:

   ```bash
   ansible-playbook -i inventory/inventory playbooks/deploy_etcd.yaml --ask-vault-pass
   ```

### Deploy Patroni/Postgres

1. Check the appropriate settings in the group_vars and vault.
2. Execute:

   ```bash
   ansible-playbook -i inventory/inventory playbooks/deploy_patroni.yaml --ask-vault-pass
   ```

### Deploy Quay, Clair and Redis

1. Check the appropriate settings in the group_vars and vault.
2. Execute:

   ```bash
   ansible-playbook -i inventory/inventory playbooks/deploy_quay.yaml --ask-vault-pass
   ```

This will start the __Quay-Config__ container and places example configurations and certificates for clair into __/mnt/clair__ and certificates for quay into __/mnt/quay/config__.

It **does NOT** initialize the Quay database or automatically configuring quay. This is done manually using the Quay-Config container.

### Next Steps

The next steps must be fullfilled in order to configured Quay and Clair

1. Perform required configuration of quay using the __Quay-Config__ webUI.
2. Optional: Download clair key and certifcate
3. Download the created Quay configuration
4. Stop the __Quay-Config__ container:

   ```bash
   systemctl stop quay-config
   ```

5. Upload the Quay configuration and extract it into /mnt/quay/config on all Quay nodes. Be sure that the naming is correct:
   1. config.yaml
   2. ssl.cert
   3. ssl.key
6. Upload the Clair configuration and key to /mnt/clair on all Clair nodes. Be sure that the naming is correct:
   1. clair.crt                                                                                │-rwxrwxrwx. 
   2. clair.key
   3. config.yaml
   4. security_scanner.pem
7. Start Quay

   ```bash
   systemctl start quay 
   ```

8. Start Clair

   ```bash
   systemctl start clair
   ```

## Sources
Without great prework of these roles, this would not have been possible

* https://github.com/kostiantyn-nemchenko/ansible-role-patroni/
* https://github.com/andrewrothstein/ansible-etcd-cluster/blob/master/tasks/main.yml
* https://github.com/kelseyhightower/etcd-production-setup
* https://github.com/atosatto/ansible-minio
