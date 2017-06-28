# Section-4

## Stand up vulnerable and non-vulnerable JBOSS servers

## Stand up Attack Host with exploit tools

## Using Attack Host to exploit

## Destroying the environment

## Introducing and Setting up Kubebot

## Introducing Target and Attack Surface

## Sending a request from Kubebot for a target company from mobile
* API server receives the request
* API server drops a message in the queue to start repo-supervisor against that company’s github and waits for it to finish
* API server drops a message in the queue to start wfuzz against that company’s main domain and waits for it to finish
* Repo-supervisor finishes running and stores the results in BQ.
* API server checks the status of repo-supervisor container and sends back a success message in the channel.
* WFUZZ finishes running and stores the results in BQ.
* API server checks the status of wfuzz container and sends back a success message in the channel.

* API server waits for both success messages from the channel. Once received, drops a message in the queue to start the special worker
* Special worker queries WFUZZ dataset from BQ for all tomcat related endpoints - /manager, /admin, /console, etc.
* If found, Special worker queries Repo-supervisor dataset from BQ for all secrets.
* Special worker then tries to bruteforce the endpoint with the secret.
* If successful, special worker sends back the response to Slack.

