# Metadata harvesting

Metadata harvesting is performed using the `geospaas_harvesting` package.
Its source code is hosted at <https://github.com/nansencenter/django-geo-spaas-harvesting>

## Configuration

The harvesting process is controlled by YAML configuration files.
The structure of those files is explained in
[the readme](https://github.com/nansencenter/django-geo-spaas-harvesting/blob/master/README.md).

## Usage

Let us try out a basic harvesting configuration.
We are going to use the [example configuration file](./resources/harvest.yml) which you can find in
the resources folder. It harvests VIIRS data from NASA's EarthData platform.

The database container must be up and running (see the [basic setup workshop](./01_setup.md)).

The following command runs the harvesting process using the example configuration file.

```
docker run -d \
--name geospaas_workshops_harvesting \
-v "$(pwd)/resources/config.yml:/etc/config.yml" \
-v "$(pwd)/resources/search.yml:/etc/search.yml" \
--network 'geospaas-workshops_default' \
-e 'GEOSPAAS_DB_HOST=geospaas-workshops_postgis_db_1' \
-e 'GEOSPAAS_DB_PORT=5432' \
-e 'GEOSPAAS_DB_NAME=geodjango' \
-e 'GEOSPAAS_DB_USER=geodjango' \
-e 'GEOSPAAS_DB_PASSWORD=geospaas123' \
nansencenter/geospaas_harvesting:3.10.0 \
-m geospaas_harvesting.cli -c /etc/config.yml harvest -s /etc/search.yml

docker logs -f geospaas_workshops_harvesting
```

The harvesting container can be stopped before it finishes using the following command:

```
docker stop geospaas_workshops_harvesting
```

Once finished or stopped, the container can be removed with the following command:

```
docker rm geospaas_workshops_harvesting
```
