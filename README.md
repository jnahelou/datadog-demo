# Datadog demo environment

Ecommerce-workshop application can be found on the official [datadog github](https://github.com/DataDog/ecommerce-workshop)

## GKE install from manifests (not using helm)

Start this tutorial by deploying a minimal GKE regional cluster. Enable cilium CNI through dataplacev2 setting.

```
gcloud beta container clusters create "dd-demo" --region "europe-west9" --num-nodes "1" --enable-ip-alias --network default --enable-dataplane-v2 
```

### DataDog Agents

Use DataDog agent and cluster-agent integration to forward metric to datadog. Manifests can be found [on the documentation](https://docs.datadoghq.com/containers/kubernetes/installation/). The full features manifest is perfect for this demo. It include metrics, logging, APM, RUM and more.

Note: If you work on a datadog account based on EU. Update each agent and cluster-agent to add a new environment variable `DD_SITE` with value `datadoghq.eu`.

Note: cluster token must be 32 chars! Otherwise, Agent will log authentication error.

### Additional integration

#### Cilium

GKE override default Cilium metric server listener. To configure DataDog agent, we will override default the configuration file for cilium integration. To do it, let's create a new configmap from the config file and mount it in folder.

You can find the default config file in the folder `/etc/datadog-agent/conf.d/cilium.d` one the DataDog agent. 
Replace `agent_endpoint: http://%%host%%:9090/metrics` by `agent_endpoint: http://%%host%%:9990/metrics`.
```
$ kubectl create configmap datadog-agent-cilium-integration --from-file=cilium_auto_conf.yaml 
$ cat datadog-agent-all-features.yaml
...
  volumeMounts:
    - name: cilium-auto-conf
      mountPath: /etc/datadog-agent/conf.d/cilium.d
volumes:
  - name: cilium-auto-conf
    configMap:
      name: datadog-agent-cilium-integration
```

Redeploy agent and check cilium integration is working using `kubectl exec -it datadog-xxxx agent status`

#### Kubernetes stats core and Tag extraction

DataDog cluster-agent provide Kubernetes configuration monitoring. The cluster-agent is already deploy in the all-features manifest. If not, follow [kubernetes_state_core](https://docs.datadoghq.com/integrations/kubernetes_state_core/) (you can get manifests from [here](https://github.com/DataDog/datadog-agent/blob/main/Dockerfiles/manifests/kubernetes_state_core)).

In all-features manifest, cluster-agent is not configured to collects K8S resources. 
Push the `manifests/all-in-one/cluster-agent-confd-configmap.yaml` config map and overide cluster-agent configuration to use this file:

The `join_standard_tags: true` enable [tag extraction](https://docs.datadoghq.com/getting_started/tagging/unified_service_tagging/?tabs=kubernetes)

### E-Commerce website

Deploy manifests available in `deploy/gcp/gke` folder (skip the datadog-agent file) from [datadog github](https://github.com/DataDog/ecommerce-workshop)

#### RUM
After creating a new RUM application, update frontend `deploy/gcp/gke/frontend.yaml` manifest to include application secrets

#### Improve DataDog Tags

DataDog is able to discover standard K8S automatically [doc](https://docs.datadoghq.com/containers/kubernetes/tag/?tabs=containerizedagent). Update deployments and pods and add the following labels:

```
  labels:
    service: xxx
    app: ecommerce
    app.kubernetes.io/version: 1.0.0
    app.kubernetes.io/name: ecommerce
    app.kubernetes.io/component: xxxxx
    tags.datadoghq.com/service: xxxxx-service
    tags.datadoghq.com/env: development
```

Don't forget to inject environment variable for SDK used in application:

```
        env:
          - name: DD_SERVICE
            valueFrom:
              fieldRef:
                fieldPath: metadata.labels['tags.datadoghq.com/service']
          - name: DD_VERSION
            valueFrom:
              fieldRef:
                fieldPath: metadata.labels['app.kubernetes.io/version']
```

You can also use `annotations` to create custom tags.

```
  annotations:
    ad.datadoghq.com/tags: '{"app": "ecommerce"}'
```

If configuration is correct, on service page, deployment and pods should be counted. On the configuration wheel, `version tagging` should be enabled.

#### Log filter

Some logs are not correctly parsed by default glog.

1. Create a new Pipeline applied on `service:(advertisements-service OR discountsservice)` name `ecommerce-app`
2. Create a Grok parser with the following rule:
```
logparsing %{date("yyyy-MM-dd HH:mm:ss,SSS"):date}\s+%{word:level}\s+\[%{word:scope}\]\s+\[%{data:file}\]\s+\[dd.service\=%{data:service}\s+dd.env\=%{word:env}\s+dd.version\=%{data:version}\s+dd.trace_id\=%{integer:trace_id}\s+dd.span_id\=%{integer:span_id}\]\s+-\s+%{data}
```
3. Remap the level attribut as status field

### Global

You can find an opinionated dashbord on this repository. Import the json file to install the dashboard.


#### Trace 

Add a retention policy to have more than 15min history. Keep all traces. If you have multiple applications on your DataDog account, add a service filter.

#### Create a monitor

Create a new monitor named `High latency on store-frontend` with the query `avg(last_5m):avg:trace.rack.request.duration{service:store-frontend} > 8`. Add the tag `app:ecommerce`.

#### Create a SLO

Create a new SLO named `[Errors] Serving 5xx class` with the query `(sum:trace.rack.request.hits.by_http_status{service:store-frontend,http.status_class:2xx}.as_count()) / (sum:trace.rack.request.hits.by_http_status{service:store-frontend AND (http.status_class:5xx OR http.status_class:2xx)}.as_count())` and the tag `app:ecommerce`.

#### Replay traffic 

Build and run the replay traffic docker image:
```
$ docker run -it --rm --env FRONTEND_HOST=34.155.45.87 --env FRONTEND_PORT=3000 traffic-replay
```

#### Usage

Default images provides various errors. To deploy working application, replace `image: ddtraining/xxxxx:latest` by `image: ddtraining/xxxxx-fixed:latest`, update the deployment version and redeploy.
