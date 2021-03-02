# hashicorp-vault-spring
Repositories contains implementation of Hashicorp Vault secret management

---

## Problem Goal
While using MYSQL as database for Sprint Boot application we use store the Database credentials in plaintext over in our source-control in the from of configuration files for application.yml, which their by increases the blast radius and hampers the overall security, in order to reduce the blast radius we can make use of Vault, which acts as secret management to store secrets.

## Prerequisite Setup
1. MYSQL Running
2. Vault Running

---
For the ease of setup we will use docker-compose to setup the Vault & MYSQl on local development environment.
We will use Docker-Compose to have these application ready, you need to install docker 
https://docs.docker.com/get-docker/


For the purpose of this example we will use MYSQL PLUGIN for Vault
https://www.vaultproject.io/docs/secrets/databases/mysql-maria

API for Vault for Databases
https://www.vaultproject.io/api/secret/databases/mysql-maria
https://www.vaultproject.io/api/secret/databases

---

## Setup

Clone this repository & run
```
docker-compose up

```
Verify if the containers are running
```
docker ps
```
- After running the command you should be able to see two docker containers running with the images as
    - vault-dev
    - mysql-dev

----


## Create the MYSQL User for the Vault
- After we have verified that the docker containers for Vault & MYSQL are running, its time to create the user.


```
docker exec -it dev-mysql mysql -h localhost -uroot -pready2go
CREATE USER 'spring-vault'@'localhost' IDENTIFIED BY 'vault';
GRANT ALL PRIVILEGES ON *.* TO 'spring-vault'@'localhost' WITH GRANT OPTION;
```

```
docker exec -it dev-mysql mysql -h localhost -uroot -pready2go
CREATE USER 'spring-vault'@'%' IDENTIFIED BY 'vault';
GRANT ALL PRIVILEGES ON *.* TO 'vault-spring'@'%' WITH GRANT OPTION;
```

```
username: vault-spring
password: vault
```



```
mysql -h localhost -usprint-vault -pready2go
```
- With the help of we have exec'd into mysql container and create a localhost user named as sprint-vault to allow permission to allow vault to create users.


## Setting up Vault
- As its local development environment, it will run vault in dev mode.

- Exec into the Vault
```
docker exec -it vault sh
```

```
You may need to set the following environment variable:

    $ export VAULT_ADDR='http://0.0.0.0:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: iXVnDx8yqQK1YRXz2nmVJ8qIKY/hXG3DlHl4VTgg2WY=
Root Token: 00000000-0000-0000-0000-000000000000

Development mode should NOT be used in production installations!
```

- Grab the Vault Root Token from the above output
```
export VAULT_TOKEN='00000000-0000-0000-0000-000000000000'
export VAULT_ADDR='http://127.0.0.1:8200'
```

- Next step is to setup enable MYSQL
```
vault secrets enable mysql
```

- 
```
vault write mysql/config/connection  connection_url="vault-spring:vault@tcp(dev-mysql:3306)/"
```


## Creating the Role
### What is Role in Vault?
- Role is used to 
Role is mapping of name in Vault to an SQL statement to execute to create the database credential.
In this we can have Static Role or Dynamic Role.

- Static Role
    - The database secrets engine supports the concept of "static roles", which are a 1-to-1 mapping of Vault Roles to usernames in a database.

- Dynamic Role
    - Dynamic Role generates a unique combination of username and password

```
vault write mysql/roles/readonly \
sql="CREATE USER 'micros-svc-1'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO 'micros-svc-1'@'%';"
```


Creating the user only if exist static Role which will create using the same name
```
vault write mysql/roles/static-user-readonly \
sql="CREATE USER IF NOT EXISTS 'static-user-1'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO 'static-user-1'@'%';"
```

Execute the read to invoke the Role which will create the new user

```

```



To Verify all of this has worked run the SELECT user FROM mysql.user to verify if the new user is created.

```
mysql> select user from mysql.user;
+---------------+
| user          |
+---------------+
| micros-svc-1  |
| static-user-1 |
| vault-spring  |
| healthchecker |
| mysql.session |
| mysql.sys     |
| root          |
| spring        |
| spring-vault  |
+---------------+
9 rows in set (0.00 sec)

```

Trying to login to the MYSQL using the credentials
```
mysql -h localhost -ustat-toke-89fa7f -p957a313e-8435-599b-9e3c-dc8d11705e66
```


### Creating a Dynamic Role for Dynamic

```
vault write mysql/roles/dynamic-user-readonly \
sql="CREATE USER IF NOT EXISTS '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';"
```

### Testing Dynamic user creation

```
vault read mysql/creds/dynamic-user-readonly
Key                Value
---                -----
lease_id           mysql/creds/dynamic-user-readonly/tSrzqlc0nxLQSmig7rtMUMFr
lease_duration     768h
lease_renewable    true
password           2236ea23-a345-1d54-7878-d0d16e3417c5
username           dyna-toke-a67a8d

/ # vault read mysql/creds/dynamic-user-readonly
Key                Value
---                -----
lease_id           mysql/creds/dynamic-user-readonly/ThaXIIZQAezbkDmfxIJ7DFDa
lease_duration     768h
lease_renewable    true
password           6d77eb66-19f3-8888-d4dc-11d458c630d4
username           dyna-toke-c4ea3a

/ # vault read mysql/creds/dynamic-user-readonly
Key                Value
---                -----
lease_id           mysql/creds/dynamic-user-readonly/Z8SmnJX71ftRZbjnrfwirXqw
lease_duration     768h
lease_renewable    true
password           1100931b-8d52-8d2f-c2f9-dc2e69134b35
username           dyna-toke-793cf1
```

- Everytime new user gets created.
```
mysql> select user,Host from mysql.user;
+------------------+-----------+
| user             | Host      |
+------------------+-----------+
| dyna-toke-a67a8d | %         |
| dyna-toke-c4ea3a | %         |
| micros-svc-1     | %         |
| static-user-1    | %         |
| vault-spring     | %         |
| healthchecker    | localhost |
| mysql.session    | localhost |
| mysql.sys        | localhost |
| root             | localhost |
| spring           | localhost |
| spring-vault     | localhost |
+------------------+-----------+
11 rows in set (0.00 sec)
```