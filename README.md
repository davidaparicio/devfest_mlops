# Devfest MLOps

## Introduction

Welcome! This workshop deals with MLOps challenges, and we will explore them while manipulating wine quality data.

We will train a model to predict if a wine will be good or not,
but more importantly we will integrate our data science pipeline into an Ops pipeline. This will allow us to train,
deploy, and monitor a model in production, so that we can concentrate on the important task of data science and
improving the performance of the model.

So follow along, while we show you how to get MLOps tasks out of the way!!

## Objectives

The objective of this lab is to run a training pipeline in VertexAI, create an endpoint,
deploy a model, and deploy a monitoring job. We will use the tensorflow extended (tfx) framework
to achieve our goal.

During this lab, you will design components for VertexAI integrating
custom tensorflow code. This will allow you to later be able to handle training, deploying,
and monitoring models on your own, on a scalable and highly available platform.

## Pre requisites
- Familiarity with TensorFlow coding
- Familiarity with Kubernetes principles
- Basic knowledge of GCP (GCS, BigQuery, Cloud Shell)
- Basic knowledge of Docker
- Understanding of ML challenges
- Knowledge of MLOps challenges
- Understanding of data project management

## Technical details

- Duration : 2 hours
- Requirements : A computer, an internet connection, and a Google account

## Workshop

The starter notebook `wine_quality_notebook.ipynb` is located at the root.

Two datasets corresponding to white and red wines are located in the `par-devfest-sfeir.wine_quality` BigQuery dataset.

### Setting things up

#### The Google Cloud Platform
- Go to the Google Cloud Platform front page in a browser
- Access the devfest-sfeir project
- Start a cloudshell
- Create a Google Cloud Storage bucket using the following command:
    
  ```
  gcloud storage buckets create gs://<USERNAME>_mlops_lab --location='europe-west1'
  ```

#### The dev environment
We strongly suggest that you use a basic Cloud Shell environment for this lab, as some tools like python, pip and gcloud CLI will already be present in the environment. Later on you can always explore the repo on your own with a locally built environment.
- Clone the following repo:
  `git clone *repo_address*`
- Run the following installs:
  ```
  pip3 install --upgrade pip
  pip3 install --upgrade "tfx[kfp]<2"
  ```

#### The pipeline image

**/!\ : Due to time constraints, the Docker image part is purely informative, you have the Dockerfile, and the commands to build the image, but the image that we are going to use for this lab has already been built.**

To start:
- `cd devfest_mlops`
- `PROJECT_ID=par-devfest-sfeir`
- Build a custom Docker image:
  `docker build -f docker/dockerfile_monitoring.dev . -t europe-west1-docker.pkg.dev/$PROJECT_ID/devfest-2022/tfx_augm:1.15.1`
- Push it to the Artefact Registry:
  `docker push europe-west1-docker.pkg.dev/$PROJECT_ID/devfest-2022/tfx_augm:1.15.1`

### Getting started
The repository you've just cloned should contain the following files:
```
|_ docker
|   dockerfile_monitoring.dev
|_ src
| |_codelab
| | |  create_pipeline.py
| | |  main.py
| | |  trainer.py
| | |  transformer.py
| | |_ monitorer_component
| |    __init__.py
| |    component.py
| |    executor.py
| |_solutions
|   |  create_pipeline.py
|   |  main.py
|   |  trainer.py
|   |  transformer.py
|   |_ monitorer_component
|       __init__.py
|       component.py
|       executor.py
|
|_ wine_quality_notebook.ipynb
```

#### Structure of the lab

This lab is divided into `codelab` and `solutions`.
Solutions contain a complete version of code that can be executed.
CodeLab contains a couple of TODOs with links to the necessary documentation for you to complete ;-)

Change to the devfest_mlops/src/codelab directory before starting: `cd devfest_mlops/src/codelab`

The TODOs can be found in:
 - [ ] create_pipeline.py

    By completing these TODOs you will understand how to link components between each other and how the flow of the pipeline is being built.
 - [ ] monitorer/component.py

    We have mostly used predefined components, but monitorer_component is a custom defined component. In this component we need to define the interface of the component by completing the MonitorerComponentSpec *class* from component.py
    The Executor inside the component and the component class have already been defined.
    *This part matters less as our custom Docker image already contains the solutions.monitorer_component allowing you to have a fully running pipeline even if you didn't finish this part.*

Nevertheless, don't hesitate to explore the rest of the code as well!

#### Running things

At any point you can launch the pipeline with the following command:
```
python3 main.py --google_cloud_project=par-devfest-sfeir --google_cloud_region=europe-west1 --dataset_id=wine_quality --wine_table=white_wine --gcs_bucket=<USERNAME>_mlops_lab --username=<USERNAME> --email=<EMAIL>
```
You can then inspect the logs, your progress, and any errors.

At the end of the logs in your terminal, the command will generate a link to the Google Cloud Platform where you will be able to follow your pipeline run interactively.

**/!\ : Once the monitoring job has been created, trying to re-launch the pipeline might throw an error at the last step.**

### Endpoint Predictions
It's possible to send a prediction request to a Vertex Endpoint either using a client or a curl command.

The tests can be performed either from Cloud Shell or from a Colab Notebook.

The default serving signature requires a TFRecord, so we will use a python client to send a request.

First, you need to install the necessary packages:
```
pip install --upgrade pip
pip install --upgrade "tfx[kfp]<2"
```

#### Querying from Cloud Shell
Write the request data manually or programmatically to a json file first, using python:

```
import json
data = {
  "instances" : [{
      "alcohol": [13.8],
      "chlorides": [0.036],
      "citric_acid": [0.0],
      "density": [0.98981],
      "fixed_acidity": [4.7],
      "free_sulfur_dioxide": [23.0],
      "ph": [3.53],
      "residual_sugar": [3.4],
      "sulphates": [0.92],
      "total_sulfur_dioxide": [134.0],
      "volatile_acidity": [0.785]
    }],
  "signature_name": "serving_raw"
}
with open("input.json", "w") as f:
  f.write(json.dumps(data))
```
And send a POST request from the Cloud Shell:
```
curl \
-X POST \
-H "Authorization: Bearer $(gcloud auth print-access-token)" \
-H "Content-Type: application/json" \
https://europe-west1-aiplatform.googleapis.com/v1/projects/par-devfest-sfeir/locations/europe-west1/endpoints/<ENDPOINT_ID>:rawPredict \
-d "@input.json"
```

#### Querying with python
If you request a prediction from Colab Notebook, you need to authorize the connection:
```
import sys
if 'google.colab' in sys.modules:
  from google.colab import auth
  auth.authenticate_user()
```
Then set the endpoint parameters:
```
ENDPOINT_ID='<TO DEFINE>'
GOOGLE_CLOUD_PROJECT = 'par-devfest-sfeir'
GOOGLE_CLOUD_REGION = 'europe-west1'
```
Define the client and the endpoint:
```
from google.cloud import aiplatform
import base64
import tensorflow as tf

client_options = {
    'api_endpoint': GOOGLE_CLOUD_REGION + '-aiplatform.googleapis.com'
    }

client = aiplatform.gapic.PredictionServiceClient(client_options=client_options)

endpoint = client.endpoint_path(
    project=GOOGLE_CLOUD_PROJECT,
    location=GOOGLE_CLOUD_REGION,
    endpoint=ENDPOINT_ID,
)
```
Define utility functions that are going to convert the request data to an endpoint consumable format:
```
def _float_feature(value):
  """Returns a float_list from a float / double."""
  return tf.train.Feature(float_list=tf.train.FloatList(value=[value]))

def serialize_example(alcohol, chlorides, citric_acid, density, fixed_acidity, free_sulfur_dioxide,
                      ph, residual_sugar, sulphates, total_sulfur_dioxide, volatile_acidity):
  """Creates a tf.train.Example message.  """

  # create a dictionary mapping the feature name to the tf.train.Example-compatible
  # data type
  feature = {
      'alcohol': _float_feature(alcohol),
      'chlorides': _float_feature(chlorides),
      'citric_acid': _float_feature(citric_acid),
      'density': _float_feature(density),
      'fixed_acidity': _float_feature(fixed_acidity),
      'free_sulfur_dioxide': _float_feature(free_sulfur_dioxide),
      'ph': _float_feature(ph),
      'residual_sugar': _float_feature(residual_sugar),
      'sulphates': _float_feature(sulphates),
      'total_sulfur_dioxide': _float_feature(total_sulfur_dioxide),
      'volatile_acidity': _float_feature(volatile_acidity),
  }

  # create a features message using tf.train.Example
  example_proto = tf.train.Example(features=tf.train.Features(feature=feature))
  return example_proto.SerializeToString()
```
Finally, send a request:
```
instances = [{
      "examples":{'b64':
      base64.b64encode(serialize_example(13.8, 0.036, 0.0, 0.98981, 4.7,
                                         23.0, 3.53, 3.4, 0.92, 134.0, 0.785)).decode()}
      }]
response = client.predict(endpoint=endpoint, instances=instances)
```

### Conclusions
Congratulations!!

You have now finished this lab, we encourage you to explore the objects generated by your pipelines:
- Click on the boxes and use the option to expand artifacts to get a sense of the steps, objects, and logs available after the run
- The artifacts can be found in Google Cloud Storage
- The models and endpoints can be seen in some Vertex AI pages
- In the pipelines page you can see your runs and other participants' run
- You can also explore the dataset using BigQuery, as well as the monitoring dataset you have created where you should be able to see the request previously sent (if you have any doubts about which is your dataset the ID of the monitoring job can be found by exploring your monitoring job from the endpoint page in the Vertex AI section of Google Cloud Platform)

Until next time!

### License
This dataset is publicly available for research. The details are described in [Cortez et al., 2009].
  Please include this citation if you plan to use this database:

  P. Cortez, A. Cerdeira, F. Almeida, T. Matos and J. Reis.
  Modeling wine preferences by data mining from physicochemical properties.
  In Decision Support Systems>, Elsevier, 47(4):547-553. ISSN: 0167-9236.

  Available at: [@Elsevier] http://dx.doi.org/10.1016/j.dss.2009.05.016
                [Pre-press (pdf)] http://www3.dsi.uminho.pt/pcortez/winequality09.pdf
                [bib] http://www3.dsi.uminho.pt/pcortez/dss09.bib

1. Title: Wine Quality

2. Sources
   Created by: Paulo Cortez (Univ. Minho), António Cerdeira, Fernando Almeida, Telmo Matos and José Reis (CVRVV) @ 2009

3. Past Usage:

  P. Cortez, A. Cerdeira, F. Almeida, T. Matos and J. Reis.
  Modeling wine preferences by data mining from physicochemical properties.
  In Decision Support Systems>, Elsevier, 47(4):547-553. ISSN: 0167-9236.

You can find the original dataset at:
https://archive.ics.uci.edu/ml/machine-learning-databases/wine-quality/
