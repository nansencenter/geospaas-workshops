# Metadata harvesting

Metadata harvesting is performed using the `geospaas_harvesting` package.
Its source code is hosted at <https://github.com/nansencenter/django-geo-spaas-harvesting>

## Configuration

The harvesting process is controlled by a YAML configuration file.
The structure of this file is explained in
[the readme](https://github.com/nansencenter/django-geo-spaas-harvesting/blob/master/README.md).
You can also find configuration examples for most harvester types in
[this file](https://github.com/nansencenter/django-geo-spaas-harvesting/blob/master/geospaas_harvesting/harvest.yml).


## Usage

Let us try out a basic harvesting configuration.
We are going to use the [example configuration file](./resources/harvest.yml) which you can find in
the resources folder. It harvests VIIRS data from PO.DAAC.

The database container must be up and running (see the [basic setup workshop](./01_setup.md)).

The following command runs the harvesting process using the example configuration file.

```
docker run -d \
--name geospaas_workshops_harvesting \
-v "$(pwd)/resources/harvest.yml:/etc/harvest.yml" \
--network 'geospaas-workshops_default' \
-e 'GEOSPAAS_DB_HOST=geospaas-workshops_postgis_db_1' \
-e 'GEOSPAAS_DB_PORT=5432' \
-e 'GEOSPAAS_DB_NAME=geodjango' \
-e 'GEOSPAAS_DB_USER=geodjango' \
-e 'GEOSPAAS_DB_PASSWORD=geospaas123' \
nansencenter/geospaas_harvesting:2.2.2 \
-m geospaas_harvesting.harvest -c /etc/harvest.yml

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

Let us harvest some data from Copernicus Scihub.
This will require adding a password in the harvesting configuration.
To avoid writing the password directly in the configuration file, we can use the `!ENV` tag.
It enables us to specify the name of an environment variable in which the parameter value is stored.

Taking this into consideration, here is an example of harvester configuration for harvesting
Sentinel-3 data from November 1st 2020, from Scihub:

```yaml
copernicus_sentinel:
  class: 'CopernicusSentinelHarvester'
  max_fetcher_threads: 30
  max_db_threads: 1
  url: 'https://scihub.copernicus.eu/apihub/search'
  time_range: ['2020-11-01', '2020-11-02']
  search_terms:
    - 'platformname:Sentinel-3 AND NOT L0'
  username: <username>
  password: !ENV 'COPERNICUS_OPEN_HUB_PASSWORD'
```

To use this, `<username>` needs to be replaced by an actual Scihub account username, and a
`COPERNICUS_OPEN_HUB_PASSWORD` environment variable must be defined.

The `search_terms` parameters accepts the same strings as Scihub's
[full text search](https://scihub.copernicus.eu/userguide/FullTextSearch).
