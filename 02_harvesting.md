# Metadata harvesting

This demonstrates how to run the geospaas_harvesting package to populate the database.
The database container must be up and running (see the [first workshop](./01_setup.md)).

```shell
docker pull nansencenter/geospaas_harvesting:latest

docker run -d \
--name geospaas_workshops_harvesting \
-v "$(pwd)/resources/harvest.yml:/etc/harvest.yml" \
--network 'geospaas-workshops_default' \
-e 'GEOSPAAS_DB_HOST=geospaas-workshops_postgis_db_1' \
-e 'GEOSPAAS_DB_PORT=5432' \
-e 'GEOSPAAS_DB_NAME=geodjango' \
-e 'GEOSPAAS_DB_USER=geodjango' \
-e 'GEOSPAAS_DB_PASSWORD=geospaas123' \
nansencenter/geospaas_harvesting:latest \
-m geospaas_harvesting.harvest -c /etc/harvest.yml

docker logs -f geospaas_workshops_harvesting
```