# Use case example

## Prerequisites

On top of the requirements in [the first workshop](./01_setup.md#prerequisites),
the following is needed:
- having an account on <https://creodias.eu/>
- installing OceanDatalab's [SEAScope](https://seascope.oceandatalab.com/) (installation instructions for Linux, MacOS and Windows are provided on the website).

## Setup

For this, you need on account on the
[Copernicus Data Space](https://dataspace.copernicus.eu/).

Write your credentials in the [cds_credentials.env](./resources/use_case/cds_credentials.env) file
with the following format:
```shell
CREODIAS_USERNAME=your_username
CREODIAS_PASSWORD=your_password
CREODIAS_TOTP_SECRET=your_totp_key
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

docker run -d \
--name geospaas_workshops_harvesting \
-v "$(pwd)/resources/config.yml:/etc/config.yml" \
-v "$(pwd)/resources/use_case/search.yml:/etc/search.yml" \
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
from geospaas.catalog.models import Dataset

# Find some data
print('Dataset.objects.count()', Dataset.objects.count())

olci_datasets = Dataset.objects.filter(source__instrument__short_name='OLCI')
slstr_datasets = Dataset.objects.filter(source__instrument__short_name='SLSTR')

print('olci_datasets.count()', olci_datasets.count())
print('slstr_datasets.count()', slstr_datasets.count())

lofoten_polygon = 'POLYGON((-5 75, 20 75, 20 65, -5 65, -5 75))'

lofoten_olci_datasets = olci_datasets.filter(geographic_location__geometry__intersects=lofoten_polygon)
lofoten_slstr_datasets = slstr_datasets.filter(geographic_location__geometry__intersects=lofoten_polygon)

results = []
for od in lofoten_olci_datasets:
    for sd in lofoten_slstr_datasets:
        # Find datasets whose spatial and temporal coverage intersects
        if (od.time_coverage_start <= sd.time_coverage_end
                and od.time_coverage_end >= sd.time_coverage_start
                and od.geographic_location.geometry.intersects(sd.geographic_location.geometry)):
            intersection = od.geographic_location.geometry.intersection(sd.geographic_location.geometry)
            if (intersection.area / od.geographic_location.geometry.area >= 0.99
                    or intersection.area / sd.geographic_location.geometry.area >= 0.99):
                results.append(od.id)
                results.append(sd.id)
print('len(results)', len(results))

# Download some interesting datasets
import geospaas_processing.tasks.core as core_tasks
import geospaas_processing.tasks.idf as idf_tasks

# download the first 10 datasets remove the [0:10] slice to download everything
for dataset_id in results[0:10]:
    core_tasks.remove_downloaded(
    idf_tasks.convert_to_idf(
    core_tasks.unarchive(
    core_tasks.download((dataset_id,)))))
```

After this, there should be new dataset folders in 
`geospaas_workshops/resources/use_case/workdir/sentinel3_olci_chl` and
`geospaas_workshops/resources/use_case/workdir/sentinel3_slstr_l2_wst`.

## Visualization in SEAScope

Let us copy these folders and the `.ini` files which accompany them to the `data/` directory of
SEAScope and start SEAScope.

If you want to adjust the visualization parameters, you can modify the `.ini` file for each
collection. The available parameters are described in the
[SEAScope user manual](https://seascope.oceandatalab.com/docs/seascope_user_manual_20190703.pdf).
