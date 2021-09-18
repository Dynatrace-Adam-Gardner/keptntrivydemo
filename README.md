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
