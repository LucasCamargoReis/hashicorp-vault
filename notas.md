course ==> https://app.pluralsight.com/library/courses/getting-started-hashicorp-vault/table-of-contents 
source ==> https://github.com/ned1313/Getting-Started-Vault

###############################
###      INSTALATION        ###
###   Open a new console    ###
###############################

VAULT_VERSION="1.0.1"
wget https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_amd64.zip

sudo apt install unzip -y
unzip vault_${VAULT_VERSION}_linux_amd64.zip
sudo chown root:root vault
sudo mv vault /usr/local/bin/   

###############################
###      Start Server       ###
###############################

vault server -dev 

export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=s.6NXTCk5ZlVk9lF0joCVapolf

vault login

###############################
###      mgmt  secret       ###
###############################

## write secret
vault kv put secret/hg2g answer=42
vault kv put secret/hg2g answer=54 ford=prefect
curl --header "X-Vault-Token: $VAULT_TOKEN" --request POST --data @secret.json $VAULT_ADDR/v1/secret/data/marvin

## get secret
vault kv get secret/hg2g
vault kv get -format=json secret/hg2g
vault kv get -format=json secret/hg2g | jq -r .data.data.answer
vault kv get -version=1 secret/hg2g
vault kv get -version=2 secret/hg2g
curl --header "X-Vault-Token: $VAULT_TOKEN" $VAULT_ADDR/v1/secret/data/marvin | jq
curl --header "X-Vault-Token: $VAULT_TOKEN" $VAULT_ADDR/v1/secret/data/hg2g?version=1 | jq .data.data

## Update
vault kv put secret/hg2g answer=54 ford=prefect
vault kv get secret/hg2g

## Soft Delete the secrets
vault kv delete secret/hg2g
vault kv get secret/hg2g
curl --header "X-Vault-Token: $VAULT_TOKEN" --request DELETE $VAULT_ADDR/v1/secret/data/hg2g

## Recovery secret affter delete
vault kv undelete -versions=2 secret/hg2g
curl --header "X-Vault-Token: $VAULT_TOKEN" --request POST   $VAULT_ADDR/v1/secret/undelete/hg2g  --data '{"versions": [2]}'

## Hard Delete the secrets - destroy
# Destroy the secrets
vault kv destroy -versions=1,2 secret/hg2g
curl --header "X-Vault-Token: $VAULT_TOKEN" --request POST   $VAULT_ADDR/v1/secret/destroy/hg2g --data '{"versions": [1,2]}'

## Remove all data about secrets
vault kv metadata delete secret/hg2g
curl --header "X-Vault-Token: $VAULT_TOKEN" --request DELETE $VAULT_ADDR/v1/secret/metadata/hg2g


###############################
###      Secret Engine      ###
###############################

# Enable a new secrets engine path
vault secrets enable -path=dev-a kv
curl --header "X-Vault-Token: $VAULT_TOKEN" --request POST  --data @secret-engine.json $VAULT_ADDR/v1/sys/mounts/dev-b

# View the secrets engine paths
vault secrets list

# Add secrets to the new secrets engine path
vault kv put dev-a/arthur love=trillian friend=ford
vault write dev-a/arthur enemy=zaphod

vault kv get dev-a/arthur
vault read dev-a/arthur

# Move the secrets engine path
vault secrets move dev-a dev-a-moved
curl --header "X-Vault-Token: $VAULT_TOKEN" --request POST --data @move-secret-engine.json $VAULT_ADDR/v1/sys/remount


###############################
###      MySQL Engine       ###
###############################

# Utilizado para criação automática de usuários e senha em um servidor Mysql.

# Enable the database secrets engine
vault secrets enable database

#### IP Publico Vault Server --> 187.74.114.161
#### IP Publico MySQL Server --> 104.208.247.127

**run on MySQLServer
# Configure MySQL roles and permissions 
## Using VM Azure bitnani

mysql -u root -p
CREATE ROLE 'dev-role';
CREATE USER 'vault'@'187.74.114.161' IDENTIFIED BY 'AsYcUdOP426i';
CREATE DATABASE devdb;
GRANT ALL ON *.* TO 'vault'@'187.74.114.161';
GRANT GRANT OPTION ON devdb.* TO 'vault'@'187.74.114.161';

**run on Vault Server
# Configure the MySQL plugin
vault write database/config/dev-mysql-database \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(104.208.247.127:3306)/" \
    allowed_roles="dev-role" \
    username="vault" \
    password="AsYcUdOP426i"

# Configure a role to be used
vault write database/roles/dev-role \
    db_name=dev-mysql-database \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT ALL ON devdb.* TO '{{name}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"

# Generate credentials on the DB from the role
vault read database/creds/dev-role

**run on MySQLServer
# Validate that the user has been created on MySQL and that the proper
SELECT User FROM mysql.user;
SHOW GRANTS FOR 'username';

# Renew the lease
vault read database/creds/dev-role
vault lease renew -increment=3600 database/creds/dev-role/6CeMd7AMafaY2VaDFSkwQzZj
vault lease renew -increment=96400 database/creds/dev-role/6CeMd7AMafaY2VaDFSkwQzZj

# Revoke the lease
vault lease revoke database/creds/dev-role/3mTIY8MWDaI7DYtDjdJWu8H5

