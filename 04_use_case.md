# Use case example

## Prerequisites

On top of the requirements in [the first workshop](./01_setup.md#prerequisites),
the following is needed:
- having an account on <https://creodias.eu/>
- installing OceanDatalab's [SEAScope](https://seascope.oceandatalab.com/) (installation instructions for Linux, MacOS and Windows are provided on the website).

## Setup

For this, you need on account on the
[Copernicus Data Space](https://dataspace.copernicus.eu/).

Write your credentials in the [credentials.env](./resources/use_case/credentials.env) file
with the following format:
```shell
COPERNICUS_DATA_SPACE_USERNAME=your_username
COPERNICUS_DATA_SPACE_PASSWORD=your_password
```

Then, just like in the first workshop, let's run the database and main containers by running the
following commands from the `geospaas_workshops` directory (even if the containers are already 
running, so that the modifications to the credentials file are taken into account).

```shell
docker-compose up -d
docker-compose logs -f main
```
Once you see the line `Quit the server with CONTROL-C.`, the containers are ready for use.

## Data harvesting

We are going to work with Sentinel 3 OLCI and SLSTR data from Copernicus Data Space.
For convenience, the harvested data can be imported in the database using the following command:

```shell
docker-compose exec main python geospaas_project/manage.py loaddata use_case_sample.json
```

<details>
<summary>To truly harvest the data, you can use the following commands.</summary>

```shell
docker rm geospaas_workshops_harvesting

# the extra slashes are here for compatibility with git bash
docker run -d \
--name geospaas_workshops_harvesting \
-v "/$(pwd)/resources/config.yml:/etc/config.yml" \
-v "/$(pwd)/resources/use_case/search.yml:/etc/search.yml" \
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

</details>

## Finding relevant data, download and convert it to IDF

Once the data is harvested, we can start looking for interesting datasets.
Then, we will download them and finally convert them to the
[IDF format](https://seascope.oceandatalab.com/docs/idf_specifications_1.5.pdf)
supported by SEAScope.

The following command opens a Django shell in the main container.

```shell
docker-compose exec main python geospaas_project/manage.py shell
```

The following code is an example of how to find relevant datasets.

```python
from datetime import timedelta
from geospaas.catalog.models import Dataset

# Find some data
print('Dataset.objects.count()', Dataset.objects.count())

s2_datasets = Dataset.objects.filter(source__platform__short_name__startswith='Sentinel-2')
s3_datasets = Dataset.objects.filter(source__platform__short_name__startswith='Sentinel-3',
                                     source__instrument__short_name='SLSTR')

print('s2_datasets.count()', s2_datasets.count())
print('s3_datasets.count()', s3_datasets.count())

split_polygon = 'POLYGON((15.78 43.9,17.32 43.17,16.42 42.35,14.89 43.25,15.78 43.9))'

split_s2_datasets = s2_datasets.filter(geographic_location__geometry__intersects=split_polygon)
split_s3_datasets = s3_datasets.filter(geographic_location__geometry__intersects=split_polygon)

print('split_s2_datasets.count()', split_s2_datasets.count())
print('split_s3_datasets.count()', split_s3_datasets.count())

# add time diff tolerance
max_time_diff = timedelta(hours=1)
results = set()
for s2d in split_s2_datasets:
    for s3d in split_s3_datasets:
        # Find datasets whose spatial and temporal coverage intersects
        if (s2d.time_coverage_start - max_time_diff <= s3d.time_coverage_end
                and s2d.time_coverage_end + max_time_diff >= s3d.time_coverage_start
                and s2d.geographic_location.geometry.intersects(s3d.geographic_location.geometry)):
            results.add(s2d)
            results.add(s3d)
print('len(results)', len(results))

# Download and convert some interesting datasets
import geospaas_processing.tasks.core as core_tasks
import geospaas_processing.tasks.idf as idf_tasks
for dataset in results:
    core_tasks.remove_downloaded(
    idf_tasks.convert_to_idf(
    core_tasks.unarchive(
    core_tasks.download((dataset.id,)))))
```

After this, there should be new dataset folders in 
`geospaas_workshops/resources/use_case/workdir/sentinel2_l1/` and
`geospaas_workshops/resources/use_case/workdir/sentinel3_slstr_sst/`.

## Visualization in SEAScope

Let us copy these folders and the `.ini` files which accompany them to the `data/` directory of
SEAScope and start SEAScope.

If you want to adjust the visualization parameters, you can modify the `.ini` file for each
collection. The available parameters are described in the
[SEAScope user manual](https://seascope.oceandatalab.com/docs/seascope_user_manual_20190703.pdf).
