# disk-level-encryption
this is a quick demo on how to setup and manage disk level encryption on a three-node multi-region cockroach database cluster

## Local Environment
We'll start by standing up a local environment with docker that will give us a source postgres database, a local storage bucket, and a cockroachdb milti-region cluster with a load balancer<br/>

But first we'll setup our encryption key and configuration options
```
mkdir -p ./settings
echo "path=/cockroach/cockroach-data,key=plain,old-key=plain" > settings/encryption-options.cfg
cockroach gen encryption-key -s 128 ./settings/master-1.key
cockroach gen encryption-key -s 128 ./settings/master-2.key
```

**Note**: make sure you start colima with ```--network-address``` and include extra resource capacity
```
mkdir -p ./data
chmod 777 ./data
colima start --network-address --memory 8 --cpu 4 --disk 100
export CRDB_VERSION=v24.1.19
docker-compose up -d
```

Next we'll need to configure our multi-region cluster
```
cockroach sql --url "postgresql://localhost:26257/defaultdb?sslmode=disable" -e """
ALTER DATABASE defaultdb SET PRIMARY REGION 'East US';
ALTER DATABASE defaultdb ADD REGION 'Central US';
ALTER DATABASE defaultdb ADD REGION 'West US';
ALTER DATABASE defaultdb SURVIVE REGION FAILURE;
"""
```


## Verify Unencrypted Data
Let's create a sample table and add some data
```
cockroach sql --url "postgresql://localhost:26257/defaultdb?sslmode=disable" -e """
CREATE TABLE test (name STRING);
INSERT INTO test (name) VALUES ('test1'), ('test2'), ('test3');
SELECT * FROM test;
"""
```
Then open a browser to http://localhost:8080 to view the dashboard for your local cockroach instance.  And http://localhost:8080/#/reports/stores/local should indicate that the data is plain text.<br/>

You can also verify the encryption status on one of the container nodes
```
docker exec -it us-east cockroach debug encryption-active-key cockroach-data
```
which should also return Plaintext: and
```
docker exec -it us-east grep -Rn "test1" cockroach-data/
```
which should return matches for our unencrypted column value


## Verify Unencrypted Data
Now we can stop our cockroach containers, update our encryption options, and restart the database.
```
docker-compose down
echo "path=/cockroach/cockroach-data,key=/cockroach/settings/master-1.key,old-key=plain" > settings/encryption-options.cfg
docker-compose up -d
```
Then open a browser to http://localhost:8080 to view the dashboard for your local cockroach instance.  And http://localhost:8080/#/reports/stores/local should indicate that the data is no longer plain text and should show the number of files that have been encrypted.<br/>

You can also verify the encryption status on one of the container nodes
```
docker exec -it us-east grep -Rn "test1" cockroach-data/
```
which should no longer return any matches for our unencrypted column value<br/>

But we'll confirm the data is still returned as plaintext from the sql console
```
cockroach sql --url "postgresql://localhost:26257/defaultdb?sslmode=disable" -e """
INSERT INTO test (name) VALUES ('test4'), ('test5'), ('test6');
SELECT * FROM test;
"""
```


## Rotate Encryption Keys
Next we can stop our cockroach containers, update our encryption keys, and restart the database.
```
docker-compose down
echo "path=/cockroach/cockroach-data,key=/cockroach/settings/master-2.key,old-key=/cockroach/settings/master-1.key" > settings/encryption-options.cfg
docker-compose up -d
```
Then open a browser to http://localhost:8080 to view the dashboard for your local cockroach instance.  And http://localhost:8080/#/reports/stores/local should indicate that both the store and data key id's have been updated.  Note this is envelope encryption and the data is not re-encryoted at this point, only the data keys, which are protected by the store key, will be re-encrypted.<br/>

But again we'll verify the encryption status one of the container nodes
```
docker exec -it us-east grep -Rn "test1" cockroach-data/
```
which still does not return any matches for our unencrypted column value<br/>

Yet still the data is returned as plaintext from the sql console
```
cockroach sql --url "postgresql://localhost:26257/defaultdb?sslmode=disable" -e """
INSERT INTO test (name) VALUES ('test7'), ('test8'), ('test9');
SELECT * FROM test;
"""
```


## Notes for Init System
We used a configuration file for our encryption options so that we did not need to modify our docker-compose file when these options change.  You can follow a similar approach if you are using something like systemd to manage your cockroach processes.  This should allow you to make changes to your encyrption options and restart cockroach from systemd without elevated privileges.  First you'll create an enviornment file with the relevant options under your user folder, i.e.
```
echo "ENTERPRISE_ENCRYPTION_OPTS=\"path=/cockroach/cockroach-data,key=/cockroach/settings/master-1.key,old-key=plain\"" > settings/encryption-options.env
```

Then you can load the environment file in your systemd cockroach service and use the ENTERPRISE_ENCRYPTION_OPTS similar to below
```
# /etc/systemd/system/cockroach.service
[Unit]
Description=CockroachDB node
After=network.target

[Service]
Type=notify
# load our encryption options
EnvironmentFile=/cockroach/settings/encryption-options.cfg

ExecStart=/cockroach/cockroach start \
  --insecure \
  --store=/cockroach/cockroach-data \
  --max-sql-memory=512MiB \
  --cache=512MiB \
  --listen-addr=us-east:26257 \
  --http-addr=0.0.0.0:8080 \
  --join=us-east:26257,us-central:26257,us-west:26257 \
  --locality=region=East US,zone=east \
  --enterprise-encryption=${ENTERPRISE_ENCRYPTION_OPTS}

Restart=on-failure
LimitNOFILE=35000

[Install]
WantedBy=multi-user.target
```