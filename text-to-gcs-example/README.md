# Overview

Demo repo to show how to bring up either standalone or distributed kafka connector for POC's. Forked from https://github.com/entechlog/. Many thanks to the maintainers of those repositories.

## Notes

- This demo has example on how to use SASL_SSL custom authentication (AWS MSK IAM auth)
- CamelCased connect variable like `defaultDataset` should be mentioned as `CONNECTOR_CAMELCASE_DEFAULT_DATASET`. This is a customization done on standalone template to add support from BQ CamelCase variables
- If you get `'bash\r'` error, please run `sudo find . -type f -exec dos2unix {} \;` to do windows to dos conversion

## How to start connector ?

- Select either standalone or distributed mode by setting KAFKA_CONNECT_MODE in the .env file
- Standalone and distributed properties files are constructed in the `configure` shell script
- This demo repo is configured to use Confluent's BigQuery connector as well as Aiven's GCS connector, but can be also updated to use any other connector. For BigQuery, create `bq-sink-connector.key` with the credential JSON for BQ
- Create copy of `.env.template` as `.env` and update the parameters specific for your environment
- Create copy of `connect.jaas.template` as `connect.jaas` and update the parameters specific for your environment
- Copy any required jar files required for CLASSPATH and update the `.env` with the correct details
- Start containers by `docker-compose up -d --build`
- If you get `'bash\r'` error, please run `sudo find . -type f -exec dos2unix {} \;` to do windows to dos conversion
- If you are running in distributed mode, you must also POST them to the Kafka Connect container. Wait for the Kafka-Connect container logs to say `Finished starting connectors and tasks`, and then POST a new connector as such:
```
curl -X POST -H "Content-Type: application/json" --data @<config file name> http://localhost:8083/connectors
```
- To delete a connector, simply run:
```
curl -X DELETE http://localhost:8083/connectors/<connector name here>
```

## To run this your own schema registry
If you would like to create your own schema registry, add in the following steps:

- Uncomment the schema registry portion of the docker-compose.yml file
- If running in standalone mode, in the `.env` file, update `SCHEMA_REGISTRY_URL=http://schema-registry:8081`. For distributed mode, update the schema registry URL parameters in your connector config json file.
- In the scripts folder, edit the shell script `generate-random-schema.sh` and change the variable `fixedVar` to 0 and set the loop to end just before the entry you need
- Run the shell script ./generate-random-schema.sh in a Visual Studio Code bash terminal (won't work in other terminals). This will create blank entries in your schema registry, which we don't need.
- Now, we will add the needed schema registry entry by manually adding it to our local schema registry in the next step
- Run the following POST command, making sure to add your relevant topic name (near the very end of the command):
```
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" --data '{<schema entry here>}' http://localhost:8081/subjects/<topic-name-here>-value/versions
```
- If running in standalone mode, you will need to restart ONLY the kafka-connect container. This is not needed for distributed mode.

## How to stop the standalone connector ?

- You can bring down the services by running `docker-compose down`

## A simple GCS bucket example

- Fill in the details of your GCP project and bucket in the file `gcs-connector.json`, including a credential file. Place this file in the secrets folder as `credentials.key`.
- Notice that all value converters in this config are "string". This way, we can easily write messages via the CLI. We have also removed compression and changed the file extension to `.txt` for easy opening of files.
- All authentication related variables in `docker-compose.yml` can be commented out.
- Create an `.env` file from `.env.template` and set `KAFKA_BOOTSTRAP_SERVERS=http://broker:9092` and `SCHEMA_REGISTRY_URL=http://schema-registry:8081`
- Run `docker-compose up -d --build` and once all containers are running, bash into the `kafka-connect` container by running this command: `docker exec -it kafka-connect bash`.
- In this container, create a topic called `test-topic` by running `kafka-topics --create --zookeeper zookeeper:2181 --topic test-topic --partitions 1 --replication-factor 1`.
- Exit the container by running the command `exit`.
- Start the GCS connector by running `curl -X POST -H "Content-Type: application/json" --data @config/sink/gcs-connector.json http://localhost:8083/connectors`.
- Bash into the `kafka-connect` container again: `docker exec -it kafka-connect bash`.
- Write a message to the topic we just created by running `kafka-console-producer --broker-list broker:9092 --topic test-topic`.
- Now type the message `hello world! This message should arrive in my GCS bucket within 30 seconds!` and press enter, then press Ctrl + C.
- You should eventually see a `.txt` file in your GCS bucket.
- When you are finished, delete the connector by running `curl -X DELETE http://localhost:8083/connectors/gcs-sink-test`.

## Reference

- https://docs.confluent.io/platform/current/tutorials/build-your-own-demos.html#standalone-mode
- https://github.com/confluentinc/cp-all-in-one/blob/6.2.1-post/cp-all-in-one-cloud/docker-compose.connect.standalone.yml
- https://github.com/entechlog/