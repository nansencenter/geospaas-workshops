# Using Jupyter notebooks

This describes how to run a Jupyter notebook with access to GeoSPaaS in a container.

## Set up

There are several needed to be able to run a Jupyter notebook from within a GeoSPaaS container:

- build a custom image based on the GeoSPaaS image, with Jupyter notebooks installed.
- modify the `docker-compose.yml` file to run a third container based on this image.

<details>
<summary>Expand to see the solution</summary>
You can find an example of Dockerfile [here](./resources/jupyter-geospaas/Dockerfile).

Add the following block to the `services` block in the `docker-compose.yml` file:

```yaml
jupyter:
  build: './resources/jupyter-geospaas/'
  depends_on: ['postgis_db']
  environment:
    PYTHONPATH: '/var/geospaas'
    GEOSPAAS_DB_HOST: 'postgis_db'
    GEOSPAAS_DB_PORT: '5432'
    GEOSPAAS_DB_NAME: 'geodjango'
    GEOSPAAS_DB_USER: 'geodjango'
    GEOSPAAS_DB_PASSWORD: 'geospaas123'
  volumes:
    - './resources/geospaas_project:/var/geospaas/geospaas_project'
  ports:
    - '8888:8888'
```
</detail>

## Usage

Once a Jupyter container is up and running, it is necessary to run the following code at the
beginning of each notebook to be able to use Django.

```python
import os
os.environ['DJANGO_SETTINGS_MODULE'] = 'geospaas_project.settings'
os.environ['DJANGO_ALLOW_ASYNC_UNSAFE'] = 'true'

import django
django.setup()
```

After that, you can use the notebook as usual.

```python
from geospaas.catalog.models import Dataset
Dataset.objects.count()
```
