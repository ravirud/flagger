# NGNIX Ingress Controller Canary Deployments

This guide shows you how to use the [Gloo](https://gloo.solo.io/) ingress controller and Flagger to automate canary deployments.

![Flagger Gloo Ingress Controller](https://raw.githubusercontent.com/weaveworks/flagger/master/docs/diagrams/flagger-gloo-overview.png)

### Prerequisites

Flagger requires a Kubernetes cluster **v1.11** or newer and Gloo ingress **1.3.5** or newer.

Install Gloo with Helm v3:

```bash
helm repo add gloo https://storage.googleapis.com/solo-public-helm
kubectl create ns gloo-system
helm upgrade -i gloo gloo/gloo \
--namespace gloo-system
```

Install Flagger and the Prometheus add-on in the same namespace as Gloo:

```bash
helm repo add flagger https://flagger.app

helm upgrade -i flagger flagger/flagger \
--namespace gloo-system \
--set prometheus.install=true \
--set meshProvider=gloo
```

Optionally you can enable Slack notifications:

```bash
helm upgrade -i flagger flagger/flagger \
--reuse-values \
--namespace gloo-system \
--set slack.url=https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK \
--set slack.channel=general \
--set slack.user=flagger
```

### Bootstrap

Flagger takes a Kubernetes deployment and optionally a horizontal pod autoscaler (HPA), 
then creates a series of objects (Kubernetes deployments, ClusterIP services and Gloo upstream groups). 
These objects expose the application outside the cluster and drive the canary analysis and promotion.

Create a test namespace:

```bash
kubectl create ns test
```

Create a deployment and a horizontal pod autoscaler:

```bash
kubectl -n test apply -k github.com/weaveworks/flagger//kustomize/podinfo
```

Deploy the load testing service to generate traffic during the canary analysis:

```bash
kubectl -n test apply -k github.com/weaveworks/flagger//kustomize/tester
```

Create an virtual service definition that references an upstream group that will be generated by Flagger
(replace `app.example.com` with your own domain):

```yaml
apiVersion: gateway.solo.io/v1
kind: VirtualService
metadata:
  name: podinfo
  namespace: test
spec:
  virtualHost:
    domains:
      - 'app.example.com'
    routes:
      - matchers:
         - prefix: /
        routeAction:
          upstreamGroup:
            name: podinfo
            namespace: test
```

Save the above resource as podinfo-virtualservice.yaml and then apply it:

```bash
kubectl apply -f ./podinfo-virtualservice.yaml
```

Create a canary custom resource (replace `app.example.com` with your own domain):

```yaml
apiVersion: flagger.app/v1alpha3
kind: Canary
metadata:
  name: podinfo
  namespace: test
spec:
  provider: gloo
  # deployment reference
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  # HPA reference (optional)
  autoscalerRef:
    apiVersion: autoscaling/v2beta1
    kind: HorizontalPodAutoscaler
    name: podinfo
  service:
    # ClusterIP port number
    port: 9898
    # container port number or name (optional)
    targetPort: 9898
  canaryAnalysis:
    # schedule interval (default 60s)
    interval: 10s
    # max number of failed metric checks before rollback
    threshold: 5
    # max traffic percentage routed to canary
    # percentage (0-100)
    maxWeight: 50
    # canary increment step
    # percentage (0-100)
    stepWeight: 5
    # Gloo Prometheus checks
    metrics:
    - name: request-success-rate
      # minimum req success rate (non 5xx responses)
      # percentage (0-100)
      threshold: 99
      interval: 1m
    - name: request-duration
      # maximum req duration P99
      # milliseconds
      threshold: 500
      interval: 30s
    # testing (optional)
    webhooks:
      - name: acceptance-test
        type: pre-rollout
        url: http://flagger-loadtester.test/
        timeout: 10s
        metadata:
          type: bash
          cmd: "curl -sd 'test' http://podinfo-canary:9898/token | grep token"
      - name: load-test
        url: http://flagger-loadtester.test/
        timeout: 5s
        metadata:
          type: cmd
          cmd: "hey -z 2m -q 5 -c 2 -host app.example.com http://gateway-proxy.gloo-system"
```

Save the above resource as podinfo-canary.yaml and then apply it:

```bash
kubectl apply -f ./podinfo-canary.yaml
```

After a couple of seconds Flagger will create the canary objects:

```bash
# applied 
deployment.apps/podinfo
horizontalpodautoscaler.autoscaling/podinfo
virtualservices.gateway.solo.io/podinfo
canary.flagger.app/podinfo

# generated 
deployment.apps/podinfo-primary
horizontalpodautoscaler.autoscaling/podinfo-primary
service/podinfo
service/podinfo-canary
service/podinfo-primary
upstreamgroups.gloo.solo.io/podinfo
```

When the bootstrap finishes Flagger will set the canary status to initialized:

```bash
kubectl -n test get canary podinfo

NAME      STATUS        WEIGHT   LASTTRANSITIONTIME
podinfo   Initialized   0        2019-05-17T08:09:51Z
```

### Automated canary promotion

Flagger implements a control loop that gradually shifts traffic to the canary while measuring key performance indicators 
like HTTP requests success rate, requests average duration and pod health.
Based on analysis of the KPIs a canary is promoted or aborted, and the analysis result is published to Slack.

![Flagger Canary Stages](https://raw.githubusercontent.com/weaveworks/flagger/master/docs/diagrams/flagger-canary-steps.png)

Trigger a canary deployment by updating the container image:

```bash
kubectl -n test set image deployment/podinfo \
podinfod=stefanprodan/podinfo:3.1.1
```

Flagger detects that the deployment revision changed and starts a new rollout:

```text
kubectl -n test describe canary/podinfo

Status:
  Canary Weight:         0
  Failed Checks:         0
  Phase:                 Succeeded
Events:
  Type     Reason  Age   From     Message
  ----     ------  ----  ----     -------
  Normal   Synced  3m    flagger  New revision detected podinfo.test
  Normal   Synced  3m    flagger  Scaling up podinfo.test
  Warning  Synced  3m    flagger  Waiting for podinfo.test rollout to finish: 0 of 1 updated replicas are available
  Normal   Synced  3m    flagger  Advance podinfo.test canary weight 5
  Normal   Synced  3m    flagger  Advance podinfo.test canary weight 10
  Normal   Synced  3m    flagger  Advance podinfo.test canary weight 15
  Normal   Synced  2m    flagger  Advance podinfo.test canary weight 20
  Normal   Synced  2m    flagger  Advance podinfo.test canary weight 25
  Normal   Synced  1m    flagger  Advance podinfo.test canary weight 30
  Normal   Synced  1m    flagger  Advance podinfo.test canary weight 35
  Normal   Synced  55s   flagger  Advance podinfo.test canary weight 40
  Normal   Synced  45s   flagger  Advance podinfo.test canary weight 45
  Normal   Synced  35s   flagger  Advance podinfo.test canary weight 50
  Normal   Synced  25s   flagger  Copying podinfo.test template spec to podinfo-primary.test
  Warning  Synced  15s   flagger  Waiting for podinfo-primary.test rollout to finish: 1 of 2 updated replicas are available
  Normal   Synced  5s    flagger  Promotion completed! Scaling down podinfo.test
```

**Note** that if you apply new changes to the deployment during the canary analysis, Flagger will restart the analysis.

You can monitor all canaries with:

```bash
watch kubectl get canaries --all-namespaces

NAMESPACE   NAME      STATUS        WEIGHT   LASTTRANSITIONTIME
test        podinfo   Progressing   15       2019-05-17T14:05:07Z
prod        frontend  Succeeded     0        2019-05-17T16:15:07Z
prod        backend   Failed        0        2019-05-17T17:05:07Z
```

### Automated rollback

During the canary analysis you can generate HTTP 500 errors and high latency to test if Flagger pauses and rolls back the faulted version.

Trigger another canary deployment:

```bash
kubectl -n test set image deployment/podinfo \
podinfod=stefanprodan/podinfo:3.1.2
```

Generate HTTP 500 errors:

```bash
watch curl -H 'Host: app.example.com' http://gateway-proxy-v2.gloo-system/status/500
```

Generate high latency:

```bash
watch curl -H 'Host: app.example.com' http://gateway-proxy-v2.gloo-system/delay/2
```

When the number of failed checks reaches the canary analysis threshold, the traffic is routed back to the primary, 
the canary is scaled to zero and the rollout is marked as failed.

```text
kubectl -n test describe canary/podinfo

Status:
  Canary Weight:         0
  Failed Checks:         10
  Phase:                 Failed
Events:
  Type     Reason  Age   From     Message
  ----     ------  ----  ----     -------
  Normal   Synced  3m    flagger  Starting canary deployment for podinfo.test
  Normal   Synced  3m    flagger  Advance podinfo.test canary weight 5
  Normal   Synced  3m    flagger  Advance podinfo.test canary weight 10
  Normal   Synced  3m    flagger  Advance podinfo.test canary weight 15
  Normal   Synced  3m    flagger  Halt podinfo.test advancement success rate 69.17% < 99%
  Normal   Synced  2m    flagger  Halt podinfo.test advancement success rate 61.39% < 99%
  Normal   Synced  2m    flagger  Halt podinfo.test advancement success rate 55.06% < 99%
  Normal   Synced  2m    flagger  Halt podinfo.test advancement success rate 47.00% < 99%
  Normal   Synced  2m    flagger  (combined from similar events): Halt podinfo.test advancement success rate 38.08% < 99%
  Warning  Synced  1m    flagger  Rolling back podinfo.test failed checks threshold reached 10
  Warning  Synced  1m    flagger  Canary failed! Scaling down podinfo.test
```

### Custom metrics

The canary analysis can be extended with Prometheus queries.

The demo app is instrumented with Prometheus so you can create a custom check that will use the HTTP request duration 
histogram to validate the canary. 

Edit the canary analysis and add the following metric:

```yaml
  canaryAnalysis:
    metrics:
    - name: "404s percentage"
      threshold: 5
      query: |
        100 - sum(
            rate(
                http_request_duration_seconds_count{
                  kubernetes_namespace="test",
                  kubernetes_pod_name=~"podinfo-[0-9a-zA-Z]+(-[0-9a-zA-Z]+)"
                  status!="404"
                }[1m]
            )
        )
        /
        sum(
            rate(
                http_request_duration_seconds_count{
                  kubernetes_namespace="test",
                  kubernetes_pod_name=~"podinfo-[0-9a-zA-Z]+(-[0-9a-zA-Z]+)"
                }[1m]
            )
        ) * 100
```

The above configuration validates the canary by checking if the HTTP 404 req/sec percentage is below 5 
percent of the total traffic. If the 404s rate reaches the 5% threshold, then the canary fails.

Trigger a canary deployment by updating the container image:

```bash
kubectl -n test set image deployment/podinfo \
podinfod=stefanprodan/podinfo:3.1.3
```

Generate 404s:

```bash
watch curl -H 'Host: app.example.com' http://gateway-proxy.gloo-system/status/400
```

Watch Flagger logs:

```
kubectl -n gloo-system logs deployment/flagger -f | jq .msg

Starting canary deployment for podinfo.test
Advance podinfo.test canary weight 5
Advance podinfo.test canary weight 10
Advance podinfo.test canary weight 15
Halt podinfo.test advancement 404s percentage 6.20 > 5
Halt podinfo.test advancement 404s percentage 6.45 > 5
Halt podinfo.test advancement 404s percentage 7.60 > 5
Halt podinfo.test advancement 404s percentage 8.69 > 5
Halt podinfo.test advancement 404s percentage 9.70 > 5
Rolling back podinfo.test failed checks threshold reached 5
Canary failed! Scaling down podinfo.test
```

If you have Slack configured, Flagger will send a notification with the reason why the canary failed.