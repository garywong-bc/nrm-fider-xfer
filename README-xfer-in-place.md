TODO
-delete builds

https://console.pathfinder.gov.bc.ca:8443/console/project/b7cg3n-tools/browse/builds/fider?tab=history


https://console.pathfinder.gov.bc.ca:8443/console/project/b7cg3n-tools/browse/builds/template-survey?tab=history

https://console.pathfinder.gov.bc.ca:8443/console/project/b7cg3n-tools/browse/builds/nrmfider-bkup?tab=history


- delete images
https://console.pathfinder.gov.bc.ca:8443/console/project/b7cg3n-tools/browse/images/fider

https://console.pathfinder.gov.bc.ca:8443/console/project/b7cg3n-tools/browse/images/fider-notls

https://console.pathfinder.gov.bc.ca:8443/console/project/b7cg3n-tools/browse/images/nrmfider-bkup



# Transferal

## Create renamed and re-tagged Fider image on target -tools namespace

```bash
oc -n b7cg3n-tools process -f ./openshift/fider-bcgov.bc.yaml | oc -n b7cg3n-tools apply -f -
oc -n b7cg3n-tools start-build nrm-feedback
```

```bash
oc -n b7cg3n-tools tag fider-bcgov:latest fider-bcgov:0.18.0
```

## Scale down source Fider

export TOOLS=b7cg3n-tools
export PROJECT=b7cg3n-deploy
export FEEDBACK=eao

To pause connections at https://${FEEDBACK}fider.pathfinder.gov.bc.ca

curl https://${FEEDBACK}fider.pathfinder.gov.bc.ca

```bash
oc -n b7cg3n-deploy scale dc ${FEEDBACK}fider-app --replicas=0
```

## Export Source DB, on Natural Resources Survey (deploy)

From your local workstation, logged into the OpenShift Console

```bash
oc -n b7cg3n-deploy rsh $(oc -n b7cg3n-deploy get pods | grep ${FEEDBACK}fider-postgresql | grep Running | awk '{print $1}')
```

In this remote shell:
```bash
cd /tmp
pg_dump -f${POSTGRESQL_DATABASE}.dump -Fc -v  ${POSTGRESQL_DATABASE}
exit
```

NOTE: Keep this file as a backup

## Transfer to intermediate location (local)

From your local workstation, logged into the OpenShift Console

```bash
cd /Users/garywong/proj/nrm-fider-xfer/db-dump
oc -n b7cg3n-deploy cp $(oc -n b7cg3n-deploy get pods | grep ${FEEDBACK}fider-postgresql | grep Running | awk '{print $1}'):/tmp/${FEEDBACK}fider.dump .
```

NOTE: Keep this file as a backup too.

## Configure intermediate location (local) DB

This ${FEEDBACK}fider file is too big to run from `oc rsh` so do it locally

Match parameters

- With the `docker-compose.yml`, then update both `.env` files to match username/password.

Reset the DB

```bash
rm -rf var/fider/pg_data/*
docker-compose up -d db
```

Once the local Docker is up, copy the DB export to a mapped volume:

```bash
cp db-dump/${FEEDBACK}fider.dump var/fider/pg_data
docker-compose exec db /bin/bash -c "pg_restore --verbose -U \${POSTGRES_USER} -d \${POSTGRES_USER} /var/lib/postgresql/data/${FEEDBACK}fider.dump"
docker-compose exec db /bin/bash -c "psql -U \${POSTGRES_USER} \${POSTGRES_DATABASE} -c \"TRUNCATE TABLE logs RESTART IDENTITY CASCADE;\" "
docker-compose exec db /bin/bash -c "pg_dump -U \${POSTGRES_USER} -f/var/lib/postgresql/data/\${POSTGRES_USER}.truncated.dump -Fc -v  \${POSTGRES_USER}"
```

On your local machine, move the truncated DB Export file and save it for backup:

```bash
cp var/fider/pg_data/${FEEDBACK}fider.truncated.dump  db-dump/
```

Verify the size difference:

```bash
ls -lah db-dump/${FEEDBACK}fider*.dump
```



## Configure Target install

### Delete old DB and app

WAIT!! Double check!  See if localhost:9999 resolves

echo "oc -n b7cg3n-deploy delete all,secret,pvc,hpa -l app=${FEEDBACK}fider"



### Deploy DB and import truncated DB export

```bash
oc -n b7cg3n-deploy new-app --file=./openshift/postgresql.dc.yaml -p FEEDBACK_NAME=${FEEDBACK}fider

oc -n b7cg3n-deploy cp db-dump/${FEEDBACK}fider.truncated.dump $(oc -n b7cg3n-deploy get pods | grep ${FEEDBACK}fider-postgresql | grep Running | awk '{print $1}'):/tmp
```

```bash
oc -n b7cg3n-deploy rsh $(oc -n b7cg3n-deploy get pods | grep ${FEEDBACK}fider-postgresql | grep Running | awk '{print $1}')
```

```bash
psql ${POSTGRESQL_DATABASE}  -c "ALTER USER ${POSTGRESQL_USER} WITH SUPERUSER"
pg_restore --verbose -U ${POSTGRESQL_USER}  -d ${POSTGRESQL_USER} /tmp/${FEEDBACK}fider.truncated.dump
psql ${POSTGRESQL_DATABASE}  -c "ALTER USER ${POSTGRESQL_USER} WITH NOSUPERUSER"
```

### Deploy App from new image

```bash
oc -n ${PROJECT} new-app --file=./openshift/fider-bcgov.dc.yaml -p FEEDBACK_NAME=${FEEDBACK}fider -p IS_NAMESPACE=${TOOLS} EMAIL_SMTP_USERNAME=Gary.T.Wong@gov.bc.ca
```

HERE


### Sign-in to verify the restore is successful

Visit `http://<targetFider>/signup/verify?k=<key>`


If the SMTP Mail is configured then click on that email link..  rsh into the target DB container:

```bash
psql -U ${POSTGRES_USER} ${POSTGRES_DATABASE}
select * from email_verifications;
```

Obtain `key` value and craft the URL to verify:
`http://<targetFider>/signup/verify?k=<key>`

#### Set new account to be Administrator

If required, rsh into the target DB container:

```bash
psql -U ${POSTGRES_USER} ${POSTGRES_DATABASE}
```

```sql
${FEEDBACK}fider=#  update users set role = 3 where name = 'Gary.T.Wong@gov.bc.ca';
```

### Truncate the Logs table

```bash
psql -U ${POSTGRES_USER} ${POSTGRES_DATABASE}
TRUNCATE TABLE logs RESTART IDENTITY CASCADE;
```

## Verify the new imported site

## Re-export the smaller DB

```bash
docker-compose exec db /bin/bash
cd /tmp
pg_dump -U ${POSTGRES_USER} -f${POSTGRES_USER}.dump -Fc -v  ${POSTGRES_USER}
```

➜  nrm-fider-xfer git:(master) ✗ docker cp 1294192fd518:/tmp/${FEEDBACK}fider.truncated.dump db-dump 

➜  nrm-fider-xfer git:(master) ✗ ls -lah db-dump
rw-r--r--   1 garywong  1839645156   257M 30 Apr 16:48 mdsfider.dump
-rw-r--r--   1 garywong  1839645156   166K 30 Apr 17:47 mdsfider.truncated.dump


## Repeat Prepare Target DB






====

Legacy

### Prepare target DB

* reset the DB before bringing up  `rm -rf var/fider/pg_data/*`
* `docker-compose up`
* let the default Fider install run (to get schemas)
* drop tables in preparation for import

```bash
psql -U ${POSTGRES_USER} ${POSTGRES_DATABASE} << EOF
DROP TABLE IF EXISTS attachments         CASCADE;
DROP TABLE IF EXISTS blobs               CASCADE;
DROP TABLE IF EXISTS comments            CASCADE;
DROP TABLE IF EXISTS email_verifications CASCADE;
DROP TABLE IF EXISTS events              CASCADE;
DROP TABLE IF EXISTS logs                CASCADE;
DROP TABLE IF EXISTS migrations_history  CASCADE;
DROP TABLE IF EXISTS notifications       CASCADE;
DROP TABLE IF EXISTS oauth_providers     CASCADE;
DROP TABLE IF EXISTS post_subscribers    CASCADE;
DROP TABLE IF EXISTS post_tags           CASCADE;
DROP TABLE IF EXISTS post_votes          CASCADE;
DROP TABLE IF EXISTS tenants_billing     CASCADE;
DROP TABLE IF EXISTS user_providers      CASCADE;
DROP TABLE IF EXISTS user_settings       CASCADE;
DROP TABLE IF EXISTS users               CASCADE;
DROP TABLE IF EXISTS tags                CASCADE;
DROP TABLE IF EXISTS posts               CASCADE;
DROP TABLE IF EXISTS tenants             CASCADE;
EOF
```


