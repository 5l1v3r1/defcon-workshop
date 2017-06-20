# Section-3

## Switching back context to Minikube

* `kubectl config use-context minikube`

## Running NMAP on the local K8S cluster

* `kubectl apply -f nmap-deployment.yaml`

## Google PubSub in action

* `virtualenv env`
* `. env/bin/activate`
* `pip install --upgrade google-cloud-pubsub`
* `gcloud config list` - Verify your account, project and active configuration are correctly setup
* `python createtopicandsub.py` - Creating the topic and subscription
* `python sendtotopic.py` - Sending the message to the topic
* `python listenfromsub.py` - Listening for that message from the subscription
* `python deletetopicandsub.py` - Deleting the topic and subscription

References:
* https://cloud.google.com/pubsub/docs/reference/libraries#client-libraries-install-python

## Convert NMAP data into BigQuery ingest-able format using a Data converter worker

## Querying BigQuery for data

## Running Cronjobs
