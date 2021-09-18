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
