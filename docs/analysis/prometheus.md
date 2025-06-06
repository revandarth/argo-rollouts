# Prometheus Metrics

A [Prometheus](https://prometheus.io/) query can be used to obtain measurements for analysis.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
  - name: service-name
  metrics:
  - name: success-rate
    interval: 5m
    # NOTE: prometheus queries return results in the form of a vector.
    # So it is common to access the index 0 of the returned array to obtain the value
    successCondition: result[0] >= 0.95
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus.example.com:9090
        # timeout is expressed in seconds
        timeout: 40
        headers:
        - key: X-Scope-OrgID
          value: tenant_a
        query: |
          sum(irate(
            istio_requests_total{reporter="source",destination_service=~"{{args.service-name}}",response_code!~"5.*"}[5m]
          )) /
          sum(irate(
            istio_requests_total{reporter="source",destination_service=~"{{args.service-name}}"}[5m]
          ))
```

The example shows Istio metrics, but you can use any kind of metric available to your prometheus instance. We suggest
you validate your [PromQL expression](https://prometheus.io/docs/prometheus/latest/querying/basics/) using the [Prometheus GUI first](https://prometheus.io/docs/introduction/first_steps/#using-the-expression-browser).

See the [Analysis Overview page](../../features/analysis) for more details on the available options.

## Range queries

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: range-query-example
spec:
  args:
  - name: service-name
  - name: lookback-duration
    value: 5m
  metrics:
  - name: success-rate
    # checks that all returned values are under 1000ms
    successCondition: "all(result, # < 1000)"
    failureLimit: 3
    provider:
      prometheus:
        rangeQuery:
          # See https://expr-lang.org/docs/language-definition#date-functions
          # for value date functions
          # The start point to query from
          start: 'now() - duration("{{args.lookback-duration}}")'
          # The end point to query to
          end: 'now()'
          # Query resolution width 
          step: 1m
        address: http://prometheus.example.com:9090
        query: http_latency_ms{service="{{args.service-name}}"}
```

### Range query and successCondition/failureCondition

Since range queries will usually return multiple values from prometheus. It is important to assert on every value returned. See the following examples:

* ❌ `result[0] < 1000` - this will only check the first value returned
* ✅ `all(result, # < 1000)` - checks every value returns from prometheus

See [expr](https://github.com/expr-lang/expr) for more expression options.

## Authorization

### Utilizing Amazon Managed Prometheus

Amazon Managed Prometheus can be used as the prometheus data source for analysis. In order to do this the namespace where your analysis is running will have to have the appropriate [IRSA attached](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-onboard-ingest-metrics-new-Prometheus.html#AMP-onboard-new-Prometheus-IRSA) to allow for prometheus queries. Once you ensure the proper permissions are in place to access AMP, you can use an AMP workspace url in your ```provider``` block and add a SigV4 config for Sigv4 signing:

```yaml
provider:
  prometheus:
    address: https://aps-workspaces.$REGION.amazonaws.com/workspaces/$WORKSPACEID
    query: |
      sum(irate(
        istio_requests_total{reporter="source",destination_service=~"{{args.service-name}}",response_code!~"5.*"}[5m]
      )) /
      sum(irate(
        istio_requests_total{reporter="source",destination_service=~"{{args.service-name}}"}[5m]
      ))
    authentication:
      sigv4:
        region: $REGION
        profile: $PROFILE
        roleArn: $ROLEARN
```

### With OAuth2

You can setup an [OAuth2 client credential](https://datatracker.ietf.org/doc/html/rfc6749#section-4.4) flow using the following values:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
  - name: service-name
  # from secret
  - name: oauthSecret  # This is the OAuth2 shared secret
    valueFrom:
      secretKeyRef:
        name: oauth-secret
        key: secret
  metrics:
  - name: success-rate
    interval: 5m
    # NOTE: prometheus queries return results in the form of a vector.
    # So it is common to access the index 0 of the returned array to obtain the value
    successCondition: result[0] >= 0.95
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus.example.com:9090
        # timeout is expressed in seconds
        timeout: 40
        authentication:
          oauth2:
            tokenUrl: https://my-oauth2-provider/token
            clientId: my-client-id
            clientSecret: "{{ args.oauthSecret }}"
            scopes: [
              "my-oauth2-scope"
            ]
        query: |
          sum(irate(
            istio_requests_total{reporter="source",destination_service=~"{{args.service-name}}",response_code!~"5.*"}[5m]
          )) /
          sum(irate(
            istio_requests_total{reporter="source",destination_service=~"{{args.service-name}}"}[5m]
          ))
```

The AnalysisRun will first get an access token using that information, and provide it as an `Authorization: Bearer` header for the metric provider call.

## Additional Metadata

Any additional metadata from the Prometheus controller, like the resolved queries after substituting the template's
arguments, etc. will appear under the `Metadata` map in the `MetricsResult` object of `AnalysisRun`.



## Skip TLS verification

You can skip the TLS verification of the prometheus host provided by setting the options `insecure: true`.

```yaml
provider:
  prometheus:
    address: https://prometheus.example.com
    insecure: true
    query: |
      sum(irate(
        istio_requests_total{reporter="source",destination_service=~"{{args.service-name}}",response_code!~"5.*"}[5m]
      )) /
      sum(irate(
        istio_requests_total{reporter="source",destination_service=~"{{args.service-name}}"}[5m]
      ))
```
