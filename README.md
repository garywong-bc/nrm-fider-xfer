# Transferal

## Create renamed and re-tagged Fider image on target -tools namespace

```bash
oc -n csnr-devops-lab-tools process -f ./openshift/fider-bcgov.xfer.bc.yaml | oc -n csnr-devops-lab-tools apply -f -
oc -n csnr-devops-lab-tools start-build nrm-feedback
```

## Scale down source MDS Fider

To pause connections at https://mdsfider.pathfinder.gov.bc.ca

```bash
oc -n csnr-devops-lab-deploy scale dc mdsfider-app --replicas=0
```

## Export Source DB, on Natural Resources Survey (deploy)

From your local workstation, logged into the OpenShift Console

```bash
oc -n b7cg3n-deploy rsh $(oc -n b7cg3n-deploy get pods | grep mdsfider-postgresql | grep Running | awk '{print $1}')
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
oc -n b7cg3n-deploy cp $(oc -n b7cg3n-deploy get pods | grep mdsfider-postgresql | grep Running | awk '{print $1}'):/tmp/mdsfider.dump .
```

NOTE: Keep this file as a backup too.

### Export DB and JWT Secrets (to match source)

```bash
oc -n b7cg3n-deploy get secret mdsfider-jwt -o yaml --export=true > mdsfider-jwt.secret.yaml
oc -n b7cg3n-deploy get secret mdsfider-postgresql -o yaml --export=true > mdsfider-postgresql.secret.yaml
```

## Configure intermediate location (local) DB

This mdsfider file is too big to run from `oc rsh` so do it locally

Match parameters
* With the `docker-compose.yml`, then update both `.env` files to match username/password.


Reset the DB

```bash
rm -rf var/fider/pg_data/*
run `docker-compose up -d db
```

Once the local Docker is up, copy the DB export to a mapped volume:

```bash
cp db-dump/mdsfider.dump var/fider/pg_data
docker-compose exec db /bin/bash
cd /var/lib/postgresql/data/
pg_restore --verbose -U ${POSTGRES_USER}  -d ${POSTGRES_USER} mdsfider.dump
psql -U ${POSTGRES_USER} ${POSTGRES_DATABASE}
TRUNCATE TABLE logs RESTART IDENTITY CASCADE;
exit
```

Thenre-export to a smaller DB export file:

```bash
pg_dump -U ${POSTGRES_USER} -f${POSTGRES_USER}.truncated.dump -Fc -v  ${POSTGRES_USER}
exit
```

On your local machine, move the truncated DB Export file and save it for backup:

```bash
cp var/fider/pg_data/mdsfider.truncated.dump  db-dump/
```

Verify the size difference:

```bash
ls -lah db-dump/mdsfider*.dump
```

## Configure Target install

### Match target DB parameters

```bash
oc -n csnr-devops-lab-deploy apply -f mdsfider-jwt.secret.yaml
oc -n csnr-devops-lab-deploy apply -f mdsfider-postgresql.secret.yaml
```

### Deploy DB and import truncated DB export

```bash
oc -n csnr-devops-lab-deploy new-app --file=./openshift/postgresql.xfer.dc.yaml -p FEEDBACK_NAME=mdsfider

oc -n csnr-devops-lab-deploy cp db-dump/mdsfider.truncated.dump $(oc -n csnr-devops-lab-deploy get pods | grep mdsfider-postgresql | grep Running | awk '{print $1}'):/tmp
```

```bash
oc -n csnr-devops-lab-deploy rsh $(oc -n csnr-devops-lab-deploy get pods | grep mdsfider-postgresql | grep Running | awk '{print $1}')
```

```bash
psql ${POSTGRESQL_DATABASE}  -c "ALTER USER ${POSTGRESQL_USER} WITH SUPERUSER"
pg_restore --verbose -U ${POSTGRESQL_USER}  -d ${POSTGRESQL_USER} /tmp/mdsfider.truncated.dump
psql ${POSTGRESQL_DATABASE}  -c "ALTER USER ${POSTGRESQL_USER} WITH NOSUPERUSER"
```

HERE

### Match app parameters

Except for route to avoid route collision with https://mdsfider.pathfinder.gov.bc.ca 


> oc -n csnr-devops-lab-deploy new-app --file=./openshift/fider-bcgov.xfer.dc.yaml -p FEEDBACK_NAME=mdsfider -p IS_NAMESPACE=csnr-devops-lab-tools EMAIL_SMTP_USERNAME=Gary.T.Wong@gov.bc.ca





TODEL
oc -n csnr-devops-lab-deploy delete dc/mdsfider-app svc/mdsfider route/mdsfider secret/mdsfider-jwt hpa/mdsfider




### Copy to local device

Once the DB pod has been deployed, copy the `.dump` file to the target filesystem.  There is no need for the Fider app to be actually set up as we'll be dropping and restoring from backup.

```bash
oc -n b7cg3n-deploy cp mdsfider-postgresql-2-5gv44:/tmp/mdsfider.dump .
```

If the final target is local Docker, then

```bash
cp db-dump/mdsfider.dump var/fider/pg_data 
```

If the final target is OpenShift, then

```bash
cd \tmp
oc -n xxx cp mdsfider.dump mdsfider-postgresql-2-5gv44:/tmp/
```

### Recreate secrets in the target namespace

```bash
oc -n empr-mds-prod create -f mdsfider-jwt.secret.yaml
oc -n empr-mds-prod create -f mdsfider-postgresql.secret.yaml
```

### Import data

If on local
```bash
docker-compose exec db /bin/bash
cd /var/lib/postgresql/data/
pg_restore --verbose -U ${POSTGRES_USER}  -d ${POSTGRES_USER} mdsfider.dump
```

If on OpenShift
```bash
pg_restore  --verbose -U ${POSTGRES_USER}  -d ${POSTGRES_USER} -f /tmp/mdsfider.dump
```

Truncate big logs table:

```bash
psql -U ${POSTGRES_USER} ${POSTGRES_DATABASE}
TRUNCATE TABLE logs RESTART IDENTITY CASCADE;
```

Re-export to a smaller DB export:

```bash
mkdir /tmp/truncated
cd /tmp/truncated
pg_dump -f${POSTGRESQL_DATABASE}.dump -Fc -v  ${POSTGRESQL_DATABASE}
```




oc project moe-gwells-<dev/test/prod>
oc get pods
oc debug < postgresql-<identifer>
In that debug shell,

cd /var/lib/pgsql/data
mv userdata userdata_broken




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
mdsfider=#  update users set role = 3 where name = 'Gary.T.Wong@gov.bc.ca';
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

➜  nrm-fider-xfer git:(master) ✗ docker cp 1294192fd518:/tmp/mdsfider.truncated.dump db-dump 

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


