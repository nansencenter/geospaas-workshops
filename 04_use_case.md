# Use case example

## Prerequisites

On top of the requirements in [the first workshop](./01_setup.md#prerequisites),
the following is needed:
- having an account on <https://creodias.eu/>
- installing OceanDatalab's [SEAScope](https://seascope.oceandatalab.com/) (installation instructions for Linux, MacOS and Windows are provided on the website).

## Setup

Just like in the first workshop, let's run the database and main containers by running the following
commands from the `geospaas_workshops` directory.

```
docker-compose up -d
docker-compose logs -f main
```
Once you see the line `Quit the server with CONTROL-C.`, the containers are ready for use.

## Data harvesting

We are going to work with Sentinel 3 OLCI and SLSTR data from Creodias.
For convenience, the harvested data can be imported in the database using the following command:

```
docker-compose exec main python geospaas_project/manage.py loaddata use_case_sample.json
```

<details>
<summary>To truly harvest the data, you can use the following commands.</summary>

```
docker pull nansencenter/geospaas_harvesting:latest

docker run -d \
--name geospaas_workshops_harvesting \
-v "$(pwd)/resources/use_case/harvest.yml:/etc/harvest.yml" \
--network 'geospaas-workshops_default' \
-e 'GEOSPAAS_DB_HOST=geospaas-workshops_postgis_db_1' \
-e 'GEOSPAAS_DB_PORT=5432' \
-e 'GEOSPAAS_DB_NAME=geodjango' \
-e 'GEOSPAAS_DB_USER=geodjango' \
-e 'GEOSPAAS_DB_PASSWORD=geospaas123' \
-e "COPERNICUS_OPEN_HUB_PASSWORD=$COPERNICUS_OPEN_HUB_PASSWORD" \
nansencenter/geospaas_harvesting:latest \
-m geospaas_harvesting.harvest -c /etc/harvest.yml

docker logs -f geospaas_workshops_harvesting
```

</details>

## Finding relevant data

Once the data is harvested, we can start looking for interesting datasets.

The following command opens a Django shell in the main container.

```
docker-compose exec main python geospaas_project/manage.py shell
```

The following code is an example of how to find relevant datasets.

```python
from geospaas.catalog.models import Dataset

Dataset.objects.count()

olci_datasets = Dataset.objects.filter(source__instrument__short_name='OLCI')
slstr_datasets = Dataset.objects.filter(source__instrument__short_name='SLSTR')

olci_datasets.count()
slstr_datasets.count()

lofoten_polygon = 'POLYGON((-5 75, 20 75, 20 65, -5 65, -5 75))'

lofoten_olci_datasets = olci_datasets.filter(geographic_location__geometry__intersects=lofoten_polygon)
lofoten_slstr_datasets = slstr_datasets.filter(geographic_location__geometry__intersects=lofoten_polygon)

for od in lofoten_olci_datasets:
    for sd in lofoten_slstr_datasets:
        # Find datasets whose spatial and temporal coverage intersects
        if (od.time_coverage_start <= sd.time_coverage_end
                and od.time_coverage_end >= sd.time_coverage_start
                and od.geographic_location.geometry.intersects(sd.geographic_location.geometry)):
            intersection = od.geographic_location.geometry.intersection(sd.geographic_location.geometry)
            if (intersection.area / od.geographic_location.geometry.area >= 0.99
                    or intersection.area / sd.geographic_location.geometry.area >= 0.99):
                print(od.id, od.entry_id)
                print(sd.id, sd.entry_id)
                print()
```

## Downloading the datasets

Let us download some of these datasets:
- go to <https://finder.creodias.eu/>
- log in if necessary
- enter the `entry_id` of the dataset you want in the "Product identifier or path" box
- click search
- click on the result in the list (there should be only one)
- click on the download button

Save the dataset files in the `geospaas_workshops/resources/use_case/` folder.

## Converting to IDF

Next, we need to convert them to the
[IDF format](https://seascope.oceandatalab.com/docs/idf_specifications_1.5.pdf)
supported by SEAScope.

The following command opens a Python shell in a container where
[geospaas_processing](https://github.com/nansencenter/django-geo-spaas-processing) is installed.

```
docker pull nansencenter/geospaas_processing_worker:latest

docker run -it --rm \
--name geospaas_workshops_conversion \
-v "$(pwd)/resources/use_case/:/var/use_case/" \
--network 'geospaas-workshops_default' \
-e 'GEOSPAAS_DB_HOST=geospaas-workshops_postgis_db_1' \
-e 'GEOSPAAS_DB_PORT=5432' \
-e 'GEOSPAAS_DB_NAME=geodjango' \
-e 'GEOSPAAS_DB_USER=geodjango' \
-e 'GEOSPAAS_DB_PASSWORD=geospaas123' \
--entrypoint python \
nansencenter/geospaas_processing_worker:latest
```

Here is the code to convert our two downloaded datasets to IDF.

```python
from geospaas_processing.converters import IDFConversionManager
conversion_manager = IDFConversionManager('/var/use_case')
conversion_manager.convert(1795, 'S3A_OL_2_WFR____20200822T121500_20200822T121800_20200822T142255_0180_062_052_1620_MAR_O_NR_002.zip')
conversion_manager.convert(5472, 'S3A_SL_2_WST____20200822T121500_20200822T121800_20200822T143516_0180_062_052_1620_MAR_O_NR_003.zip')
```

After this, there should be new dataset folders in 
`geospaas_workshops/resources/use_case/sentinel3_olci_l2_wfr` and
`geospaas_workshops/resources/use_case/sentinel3_slstr_l2_wst`.

## Visualization in SEAScope

Let us copy these folders to the `data/` directory of SEAScope and start SEAScope.

If you want to adjust the visualization parameters, you can modify the `config.ini` file in each
collection directory. The available parameters are described in the
[SEAScope user manual](https://seascope.oceandatalab.com/docs/seascope_user_manual_20190703.pdf).
