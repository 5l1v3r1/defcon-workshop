# Section-3

## Table of Contents

<!-- TOC -->

- [Section-3](#section-3)
    - [Table of Contents](#table-of-contents)
    - [Overview](#overview)
    - [Switching back context to Minikube](#switching-back-context-to-minikube)
    - [Running NMAP on the local K8S cluster](#running-nmap-on-the-local-k8s-cluster)
    - [Google PubSub in action](#google-pubsub-in-action)
    - [Convert NMAP data into BigQuery ingest-able format using a Data Converter](#convert-nmap-data-into-bigquery-ingest-able-format-using-a-data-converter)
        - [Running Locally](#running-locally)
        - [Running on a K8S cluster](#running-on-a-k8s-cluster)
    - [Querying BigQuery](#querying-bigquery)
    - [Running Cronjobs](#running-cronjobs)
    - [Cleanup](#cleanup)

<!-- /TOC -->

## Overview
In this section,
1. We will switch our `kubectl` context back from the remote K8S cluster (on GCP) to Minikube.
2. We will run a `NMAP` deployment in our local K8S cluster inside Minikube.
3. We will show an example of how `topics` and `subscriptions` work in Google PubSub.
4. We will convert NMAP data into a format that can be ingested into Google BigQuery using a custom Data Converter.
5. We will show how to run queries on Google BigQuery.
6. We will run a cronjob by running NMAP at a specific schedule against a target.
7. We will cleanup everything we setup above.


## Switching back context to Minikube

* `kubectl config use-context minikube`

## Running NMAP on the local K8S cluster

* `kubectl apply -f deployments/nmap-deployment.yaml`

## Google PubSub in action
Follow the steps in the following link to set up GOOGLE_APPLICATION_CREDENTIALS: 
* https://developers.google.com/identity/protocols/application-default-credentials

1. `virtualenv env`
2. `. env/bin/activate`
3. `pip install --upgrade google-cloud-pubsub`
4. `gcloud config list` - Verify your account, project and active configuration are correctly setup
5. `python scripts/createtopicandsub.py` - Creating the topic and subscription
6. `python scripts/sendtotopic.py` - Sending the message to the topic
7. `python scripts/listenfromsub.py` - Listening for that message from the subscription
8. `python scripts/deletetopicandsub.py` - Deleting the topic and subscription

References:
* https://cloud.google.com/pubsub/docs/reference/libraries#client-libraries-install-python

## Convert NMAP data into BigQuery ingest-able format using a Data Converter

### Running Locally
1. Create a BigQuery dataset `nmapds` and an empty table `nmap` in Google BigQuery from the GCP UI to store the processed nmap results with the following schema (all nullable):
```
ip:string
fqdn:string
port:string
protocol:string
service:string
version:string
```
2. `nmap -Pn -p 1-1000 -oN google_results.nmap google.com` - running nmap locally
* Complete the `.env.sample` file in the `data-converter` folder with the appropriate `PROJECT_ID`, `DATASET_NAME` and `TABLE_NAME` and rename it to `.env`
3. In that folder, type `go get cloud.google.com/go/bigquery` and then `go get github.com/subosito/gotenv` and finally `go run dataconvert.go ../google_results.nmap` - Run the data converter locally

### Running on a K8S cluster
1. Navigate to `IAM & Admin` -> `Service Accounts`. Create a key for the default Compute Engine Service Account and download the JSON key
2. `kubectl create secret generic googlesecret --from-file=$(CREDS_FILEPATH)` - Create a secret with the value of the secret being the JSON credentials file downloaded above. We need this because the containers on the cluster need to authenticate to our K8S cluster to be able to create anything. We don't do this locally because our gcloud environment, by default, is already configured when we first set it up but we need it when running on a K8S cluster
3. `kubectl get secrets` - Verify the secret was created
4. Make sure the environment values in the `deployments/nmap-bq-pod.yaml` deployment file are accurate. The following values need to be changed - [PROJECT_ID](https://github.com/devsecops/defcon-workshop/blob/master/section-3/deployments/nmap-bq-pod.yaml#L14) and [GOOGLE_APPLICATION_CREDENTIALS](https://github.com/devsecops/defcon-workshop/blob/master/section-3/deployments/nmap-bq-pod.yaml#L20)
5. `kubectl apply -f deployments/nmap-bq-pod.yaml`

References:
* https://github.com/maaaaz/nmaptocsv

## Querying BigQuery

* Run the below query:
```
SELECT ip, port FROM nmapds.nmap
WHERE ip IS NOT NULL AND port IS NOT NULL
GROUP BY ip, port
```

## Running Cronjobs

1. `kubectl apply -f deployments/nmap-cronjob.yaml --validate=false` - Start the cronjob
2. `kubectl get cronjobs --watch` - Watch the status of the cronjob

## Cleanup
1. `kubectl delete secret googlesecret`
2. Delete the BigQuery dataset and table
3. `kubectl delete pods --all`
4. `kubectl delete deployments --all`
5. `kubectl delete cronjobs --all`
6. `kubectl delete jobs --all`
†