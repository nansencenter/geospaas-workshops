# Basic setup

## Prerequisites

The following software needs to be installed:
- [Docker](https://docs.docker.com/get-docker/)
- [docker-compose](https://docs.docker.com/compose/install/)
- [git](https://git-scm.com/downloads)

## Running a GeoSPaaS container

First, let's clone the following git repository:
<https://github.com/nansencenter/geospaas-workshops.git>

```
git clone https://github.com/aperrin66/geospaas-workshops.git
```

Then, let's open a CLI in the `geospaas-workshops/` folder and run the following commands.
This will run two containers: a Postgis database and a container with the core GeoSPaaS code.

```
docker-compose pull
docker-compose up -d
```

The starting operation can be followed using the "logs" command from docker-compose:

```
docker-compose logs -f main
```

Once you see the line `Quit the server with CONTROL-C.`, the container is ready for use.
It exposes the basic GeoSPaaS web interface on port 8001.
You can access it by opening the following URL in your browser: <http://localhost:8001>

For now, the database is empty. Let's initialize it with sample data by running:

```
docker-compose exec main python geospaas_project/manage.py loaddata sample.json
```

The following command stops the containers:

```
docker-compose down
```

## Create a Dataset object

Let us create a dataset object in the database by hand.

We can now run an interactive Django shell in the main container.
This will enable us to interact with the GeoSPaaS database using Python code.

```
docker-compose exec main python geospaas_project/manage.py shell 
```

```python
from datetime import datetime, timezone

import geospaas.catalog.models as gcm
import geospaas.catalog.managers as managers
import geospaas.vocabularies.models as gvm

# Create a Dataset Python object
my_dataset = gcm.Dataset(
    entry_title='My dataset',
    ISO_topic_category=gvm.ISOTopicCategory.objects.get(name='Oceans'),
    data_center=gvm.DataCenter.objects.get(short_name='NERSC'),
    summary='This dataset contains some data.',
    source=gcm.Source.objects.get_or_create(
        platform=gvm.Platform.objects.get(short_name='OPERATIONAL MODELS'),
        instrument=gvm.Instrument.objects.get(short_name='Computer'))[0],
    time_coverage_start=datetime(2021, 2, 4, 10, tzinfo=timezone.utc),
    time_coverage_end=datetime(2021, 2, 4, 11, tzinfo=timezone.utc),
    geographic_location=gcm.GeographicLocation.objects.get_or_create(
        geometry='POLYGON((-34.2 55.1,-18.9 47.1,-44.4 39.7,-34.2 55.1))')[0],
    gcmd_location=gvm.Location.objects.get(type='SEA SURFACE'),
)

gcm.Dataset.objects.count()

# Save the Dataset object to the database
my_dataset.save()

gcm.Dataset.objects.count()

# Add parameter information to the dataset
my_dataset.parameters.set(gvm.Parameter.objects.filter(short_name='SST'))

# Create a URI associated with the dataset
my_dataset_uri = gcm.DatasetURI(
    uri='/foo/dataset.nc',
    dataset = my_dataset,
    service = managers.LOCAL_FILE_SERVICE,
    name = managers.FILE_SERVICE_NAME
)
my_dataset_uri.save()
```

You can now see the spatial coverage of the dataset in the Web viewer at <http://localhost:8001>
(you can filter on the source, for example).

## Basic dataset operations

In this section, there are a few examples of dataset manipulation. Let's open a Django shell.

```
docker-compose exec main python geospaas_project/manage.py shell
```

The syntax to make queries using Django is 
[here](https://docs.djangoproject.com/en/3.0/topics/db/queries/).

The list of common field lookups is
[here](https://docs.djangoproject.com/en/3.0/ref/models/querysets/#field-lookups).

There are specific spatial lookups which can be found
[here](https://docs.djangoproject.com/en/3.1/ref/contrib/gis/geoquerysets/).

Here are a few examples:

```python
from geospaas.catalog.models import Dataset

# Select the Sentinel-3 datasets
sentinel3_datasets = Dataset.objects.filter(source__platform__series_entity='Sentinel-3')

# Count the Sentinel-3 datasets
sentinel3_datasets.count()

# Define a function that prints a dataset's properties
def print_dataset(dataset):
    print('entry_id', dataset.entry_id)
    print('entry_title', dataset.entry_title)
    print('parameters', dataset.parameters)
    print('ISO_topic_category', dataset.ISO_topic_category)
    print('data_center', dataset.data_center)
    print('summary', dataset.summary)
    print('platform', dataset.source.platform)
    print('instrument', dataset.source.instrument)
    print('time_coverage_start', dataset.time_coverage_start)
    print('time_coverage_end', dataset.time_coverage_end)
    print('geographic_location', dataset.geographic_location.geometry)
    print('gcmd_location', dataset.gcmd_location)
    print('access_constraints', dataset.access_constraints)
    print('urls:')
    for uri in dataset.dataseturi_set.all():
        print(f"    {uri.uri}")

# Print the properties of one of the Sentinel-3 datasets
print_dataset(sentinel3_datasets[0])

# Define a zone of interest
zone = 'POLYGON((1 74.3,30.4 75.7,11.8 67.8,1 74.3))'

# Find datasets which intersect this rectangle
zone_datasets = Dataset.objects.filter(geographic_location__geometry__intersects=zone)
zone_datasets.count()

# Find Sentinel 3 OLCI datasets which intersect this rectangle
olci_zone_datasets = sentinel3_datasets.filter(
    source__instrument__short_name='OLCI',
    geographic_location__geometry__intersects=zone
)
olci_zone_datasets.count()

# Find Sentinel 1 datasets which intersect this rectangle
s1_zone_datasets = Dataset.objects.filter(
    source__platform__series_entity='Sentinel-1',
    geographic_location__geometry__intersects=zone
)
s1_zone_datasets.count()

# Find the Sentinel 1 and 3 OLCI datasets which cover Iceland at a given time
from datetime import datetime, timezone
start = datetime(2023, 6, 1, 15, tzinfo=timezone.utc)
stop = datetime(2023, 6, 1, 18, tzinfo=timezone.utc)

print(olci_zone_datasets.filter(time_coverage_start__lte=stop, time_coverage_end__gt=start).count())
print(s1_zone_datasets.filter(time_coverage_start__lte=stop, time_coverage_end__gt=start).count())
```
