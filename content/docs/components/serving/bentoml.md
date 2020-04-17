+++
title = "BentoML"
description = "Model serving with BentoML"
weight = 51
+++

{{% stable-status %}}

## Serving a model

### Prerequisites

1. a Kubernetes cluster

2. Docker and Docker Hub properly installed and configured in your local machine.


MAYBE REMOVE THIS
The following resources will be created in this guide.

- A Kubernetes service
- A Kubernetes deployment
- A model archived by BentoML in your local machine.

### Train and save iris classifier model with BentoML

Save the following code to a new file named `iris_classifier.py`:

```python
from bentoml import env, artifacts, api, BentoService
from bentoml.handlers import DataframeHandler
from bentoml.artifact import SklearnModelArtifact

@env(auto_pip_dependencies=True)
@artifacts([SklearnModelArtifact('model')])
class IrisClassifier(BentoService):

    @api(DataframeHandler)
    def predict(self, df):
        return self.artifacts.model.predict(df)
```

The above code defines a prediction service that requires a scikit-learn model, and asks
BentoML to figure out the required PyPI pip packages automatically. It also defined an
API, which is the entry point for accessing this prediction service. And the API is
expecting a `pandas.DataFrame` object as its input data.

Run the following code to train a classifier model and save it with BentoML.

```python
from sklearn import svm
from sklearn import datasets

from iris_classifier import IrisClassifier

if __name__ == "__main__":
    # Load training data
    iris = datasets.load_iris()
    X, y = iris.data, iris.target

    # Model Training
    clf = svm.SVC(gamma='scale')
    clf.fit(X, y)

    # Create a iris classifier service instance
    iris_classifier_service = IrisClassifier()

    # Pack the newly trained model artifact
    iris_classifier_service.pack('model', clf)

    # Save the prediction service to disk for model serving
    saved_path = iris_classifier_service.save()
```

*Use BentoML CLI to verify is the saved model:*

```shell
bentoml get IrisClassifier:latest
```

### Build and push an iris classifier model image

BentoML generates a Dockerfile for prediction service when saving the model.


```shell
saved_path=$(bentoml get IrisClassifier:latest -q | jq -r ".uri.uri")

# Replace `DOCKER_USERNAME` with the Docker Hub username.
docker_username=DOCKER_USERNAME

docker build -t $docker_username/iris-classifier $saved_path
docker push $docker_username/iris-classifier
```

### Deploy to Kubernetes

Replace the `{docker_username}` with your Docker Hub username and save the code to
a file called `iris-classifier.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: iris-classifier
  name: iris-classifier
  namespace: kubeflow
spec:
  ports:
  - name: predict
    port: 5000
    targetPort: 5000
  selector:
    app: iris-classifier
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: iris-classifier
  name: iris-classifier
  namespace: kubeflow
spec:
  selector:
    matchLabels:
      app: iris-classifier
  template:
    metadata:
      labels:
        app: iris-classifier
    spec:
      containers:
      - image: {docker_username}/iris-classifier
        imagePullPolicy: IfNotPresent
        name: iris-classifier
        ports:
        - containerPort: 5000
```

```shell
kubectl apply -f iris-classifier.yaml
```

Verify service and deployment is applied by using `kubectl` command:

```shell
kubectl get deployment --namespace kubeflow
```

### Sending prediction request

```shell
kubectl get svc iris-classifier --namespace kubeflow
```

And then send the request:

```shell
curl -i \
  --header "Content-Type: application/json" \
  --request POST \
  --data '[[5.1, 3.5, 1.4, 0.2]]' \
  http://EXTERNAL_IP:NODE_PORT/predict
```

## Monitoring metrics with Prometheus

### Perquisites

- Prometheus installed in the cluster
  - [Prometheus documentation](https://prometheus.io/docs/introduction/overview/)
  - [Installation instruction with Helm chart](https://github.com/helm/charts/tree/master/stable/prometheus)

BentoML provides Prometheus metrics endpoint out of the box for the prediction
server. BentoML provides the essential metrics and the ability to customize base on
needs.

To allow Prometheus monitoring deployed prediction server, update the deployment
template spec with annotations for Prometheus monitoring.

Change the deployment spec as follows:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: iris-classifier
  name: iris-classifier
  namespace: kubeflow
spec:
  selector:
    matchLabels:
      app: iris-classifier
  template:
    metadata:
      labels:
        app: iris-classifier
      annotations:
        prometheus.io/scrape: true
        prometheus.io/port: 5000
    spec:
      containers:
      - image: {docker_username}/iris-classifier
        imagePullPolicy: IfNotPresent
        name: iris-classifier
        ports:
        - containerPort: 5000
```

Apply the change with `kubectl` CLI.

```shell
kubectl apply -f iris-classifier.yaml
```


## Remove deployment

```shell
kubectl delete -f iris-classifier.yaml
```