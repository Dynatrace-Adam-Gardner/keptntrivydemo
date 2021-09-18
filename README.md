# Keptn Trivy Demo

## Install and Gather Details

- Install Keptn
- Install `dynatrace-service`
- Install `job-executor-service`
- Create uninitialised Git repo
- Generate a GitHub Personal Access Token with full `repo` scope

Gather details:
- Dynatrace tenant ID: `abc123.live.dynatrace.com`
- Dynatrace API token: `dtc01.***`
- Keptn Bridge Username: `default: keptn or see below`
- Keptn Bridge password: `see below`
- Keptn API URL: `see below`
- Keptn API Token: `see below`


## Download and Auth Keptn CLI

```
curl -sL https://get.keptn.sh | bash
```

Grab bridge login details:

Keptn Bridge and API URLs:
```
KEPTN_BRIDGE_URL=https://$(kubectl get service/api-gateway-nginx -n keptn -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/bridge
KEPTN_API_URL=https://$(kubectl get service/api-gateway-nginx -n keptn -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/api
```

Keptn Bridge Username:
Default is `keptn` or:
```
kubectl get secret -n keptn bridge-credentials -o jsonpath="{.data.BASIC_AUTH_USERNAME}" | base64 --decode
```

Keptn Bridge Password:
```
kubectl get secret -n keptn bridge-credentials -o jsonpath="{.data.BASIC_AUTH_PASSWORD}" | base64 --decode
```

Keptn API Token:
```
KEPTN_API_TOKEN=$(kubectl get secret keptn-api-token -n keptn -ojsonpath='{.data.keptn-api-token}' | base64 --decode)
```

Keptn auth command is available in the bridge or:
```
keptn auth --endpoint=$KEPTN_API_URL --api-token=$KEPTN_API_TOKEN
```

Hint: Once authenticated, you can retrieve at any time with: `keptn configure bridge --output`

## Create Keptn Secret:
```
keptn create secret dynatrace-api-token \
--from-literal="DT_TENANT=abc123.live.dynatrace.com" \
--from-literal="DT_API_TOKEN=dt0c01.***" \
--from-literal="KEPTN_API_URL=http://1.2.3.4/api" \
--from-literal="KEPTN_API_TOKEN=***"
```

## Create Shipyard File
```
---
apiVersion: spec.keptn.sh/0.2.0
kind: Shipyard
metadata:
  name: myshipyard
spec:
  stages:
    - name: main
      sequences:
        - name: demosequence
          tasks:
          - name: securityscan
          - name: evaluation
```

## Create a keptn project:
```
keptn create project trivyintegration \
--shipyard=shipyard.yaml \
--git-user=YOUR-GIT-USERNAME \
--git-remote-url=https://github.com/YOUR-GIT-USERNAME/YOUR-GIT-REPO \
--git-token=YOUR-GIT-PAT-TOKEN
```

## Create a new Keptn Service:
```
keptn create service trivyservice --project=trivyintegration
```

## Setup Job Executor Service
Switch to the `main` branch and create a new folder called `job` under the `trivyservice` folder.
Inside the `job` folder, create a file called `config.yaml`:

```
---
apiVersion: v2
actions:
  - name: "Run Aquasec Trivy"
    events:
      - name: "sh.keptn.event.securityscan.triggered"
    tasks:
      - name: "Run Trivy Evaluation"
        image: "adamgardnerdt/trivy:v1.0"
        env:
          - name: TRIVY_SECURITY
            valueFrom: event
            value: "$.data.scan.level"
          - name: IMAGE
            valueFrom: event
            value: "$.data.scan.image"
          - name: TAG
            valueFrom: event
            value: "$.data.scan.tag"
          - name: METRICS_ENDPOINT
            valueFrom: event
            value: "$.data.scan.metrics_endpoint"
          - name: METRICS_API_TOKEN
            valueFrom: event
            value: "$.data.scan.metrics_token"
```

This file tells the `job-executor-service` to listen for the `sh.keptn.event.securityscan.triggered` event and when heard, run the `adamgardnerdt/trivy:v1.0` container and also set some environment variables.

We will pass these environment variables in via the HTTP POST when we trigger Keptn.

![image](https://user-images.githubusercontent.com/13639658/133870539-fcb7b394-06ba-4862-9533-a83446df6d9f.png)


## Setup SLI (Metric) Retrieval

The trivy container is compatible with Dynatrace so it will push metrics into a Dynatrace backend.

Therefore we need to tell Keptn to retrieve metrics from Dynatrace. WE've done the first bits by installing the `dynatrace-service` and creating the secret. Tell the `dynatrace-service` how to auth to your DT tenant.

In the `main` branch, create a folder called `dynatrace` and inside here, create a file called `dynatrace.conf.yaml`:

```
---
dtCreds: dynatrace-api-token
```

This tells Keptn to use the secret you created earlier.

![image](https://user-images.githubusercontent.com/13639658/133870553-a70bd376-ccba-4585-81f2-4e94f698e036.png)

## Setup SLIs

Create a second file in the `dynatrace` folder called `sli.yaml`. This is how we define what metrics to pull out of Dynatrace.

This metrics ID is created automatically when the container image runs and pushes metrics to Dynatrace.

If you're using a different severity, just change `CRITICAL` to whatever level you want (case sensitive)!


```
---
indicators:
  trivy_vulns: "metricSelector=trivy.vulnerabilities.CRITICAL"
```

## Setup SLOs

Move back one directory to the `trivyservice` folder and create a file called `slo.yaml`. This will hold your SLO definitions that Keptn will use once it has pulled the metrics.

In our case, tell KEptn that as long as we have fewer than 10 CRITICAL vulnerabilities, the metric is a pass.

```
spec_version: "0.1.0"
filter: {}
comparison:
  compare_with: single_result
  include_result_with_score: pass
  number_of_comparison_results: 1
  aggregate_function: avg
objectives:
- sli: trivy_vulns
  pass:
    - criteria:
      - "<10"
total_score:
  pass: 90%
  warning: 75%
```

![image](https://user-images.githubusercontent.com/13639658/133870693-44535038-06cd-4f50-b982-b2955f112b3b.png)


## Request an Evaluation

We're ready to run an evaluation. Remember that our sequence is called `demosequence` because that's what we specified in the `shipyard.yaml` file.

So now ask Keptn to run the `demosequence`.

```
HTTP POST to http://keptn-ip/api/v1/event
{
  "data": {
    "project": "trivyintegration",
    "service": "trivyservice",
    "stage": "main",
    "labels": {
        "runBy": "AG"
    },
    "scan": {
      "level": "CRITICAL",
      "image": "mysql",
      "tag": "8.0.26",
      "metrics_endpoint": "https://abc123.live.dynatrace.com",
      "metrics_token": "dt0c01.***"
    },
    "evaluation": {
        "timeframe": "5m"
    }
  },
  "source": "POSTMAN",
  "specversion": "1.0",
  "shkeptnspecversion": "0.2.3",
  "contenttype": "application/json",
  "type": "sh.keptn.event.main.demosequence.triggered"
}
```

