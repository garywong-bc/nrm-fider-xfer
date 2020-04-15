# Transferal

## Export from source Fider install on OpenShift

`rsh` into the DB pod, or via OpenShift Console GUI:

```bash
cd \tmp
pg_dump -f${POSTGRESQL_DATABASE}.dump -Fc -v  ${POSTGRESQL_DATABASE}
```

## Copy to local device

```bash
oc -n b7cg3n-deploy cp mdsfider-postgresql-2-5gv44:/tmp/mdsfider.dump .
```

If the final target is local Docker, then

```bash
cd \tmp
cp db-dump/mdsfider.dump var/fider/pg_data
```


If the final target is OpenShift, then

```bash
cd \tmp
oc -n b7cg3n-deploy cp mdsfider.dump mdsfider-postgresql-2-5gv44:/tmp/
```

## Configure target install

### Match target DB parameters

If `docker-compose.ymnl`, then update both `.env` files to match username/password.

If the final target is OpenShift, then update the Secret.

### Prepare target DB

* run the default Fider install
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

### Import data

```bash
pg_restore --verbose -U ${POSTGRES_USER}  -d ${POSTGRES_USER} mdsfider.dump
```

### Sign-in to verify the restore is successful

If the SMTP Mail is configured then click on that email link.. otherwise:

```bash
psql -U ${POSTGRES_USER} ${POSTGRES_DATABASE}
select * from email_verifications;
```

Obtain `key` value and craft the URL to verify:
`http://<targetFider>/signup/verify?k=<key>`

#### Set new account to be Administrator

If required, login via `psql`:

```sql
mdsfider=#  update users set role = 3 where name = 'Gary.T.Wong@gov.bc.ca';
```

### Truncate the Logs table

```bash
psql -U ${POSTGRES_USER} ${POSTGRES_DATABASE}
TRUNCATE TABLE logs RESTART IDENTITY CASCADE;
```

## Verify the new imported site