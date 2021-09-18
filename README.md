# Keptn Trivy Demo

- Install Keptn
- Install `dynatrace-service`
- Install `job-executor-service`
- Create uninitialised Git repo

Gather details:
- Dynatrace tenant ID: `abc123.live.dynatrace.com`
- Dynatrace API token: `dtc01.***`
- Keptn API URL: `http://1.2.3.4/api`
- Keptn API Token: `get from the bridge`

Create a keptn secret:
```
keptn create secret dynatrace-api-token \
--from-literal="DT_TENANT=abc123.live.dynatrace.com" \
--from-literal="DT_API_TOKEN=dt0c01.***" \
--from-literal="KEPTN_API_URL=http://1.2.3.4/api" \
--from-literal="KEPTN_API_TOKEN=***"
```

Create shipyard:

```
```
