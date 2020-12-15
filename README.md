# Prometheus Federation



## Overview

### Summary

This is the beginning of a project whose purpose will be to enable a comparative look at the resource usage of individual Openshift components, by version, in order to identify targets for optimization.  This readme is primarily a walk through of the system in its current state and is strictly a PoC and visualization of the end goal.  Its architecture is subject to change.

### Goal

In order to make Openshift Container Platform competitive in the edge computing space, the platform’s resource demands need to be drastically reduced to accommodate small, unscalable hardware stacks commonly seen in the industry. We want to provide OCP engineering with a tool that will generate periodic reports of resource usage per OCP component, by build. In addition, we want to also to provide a side by side comparison with a competing Kubernetes distribution as a baseline against which we can measure progress.

### Architecture

// TODO diagram

This project consists of 1 central "hub" cluster and 1 or more experiment clusters.  We'll be making use of Prometheus federation to stream metrics from many child clusters back to a hub cluster.  On the hub, we'll be deploying Thanos and Grafana to ingest and present data, per some PromQL queries.  On each experimental cluster, we'll be installing a namespace scoped Prometheus operator from the Operator hub and deploying a Prometheus instance.  This Prometheus instance will behave like a reverse proxy - it will scrape a subset of cluster metrics from the openshift-monitoring Prometheus `/federate` endpoint and stream them to the public Thanos endpoint in the hub cluster, where it will be persisted in an S3 bucket.

## Try It Out

> This recipe is derived from the Openshift blog on [Prometheus federation](https://www.openshift.com/blog/federated-prometheus-with-thanos-receive).

### Prereqs

- At least **Three** Openshift v4.x clusters.

- An S3 bucket

### Setup

The config files under `prometheus-federation/manifests/...` are required for deploying this system. Clone this repo to your local environment

`$git clone https://github.com/redhat-et/prometheus-federation.git`

Once the clusters are deployed, I find it helpful to rename the contexts for convenience. E.g.
`$ oc config rename-context [api-HUB-CLUSTER-NAME:6443] hub`

`$ oc config rename-context [api-TEST-CLUSTER-NAME:6443] test1`

### Configuring the Hub Cluster

#### Deploy Thanos

Thanos is a modular storage interface for Prometheus - we'll be deploying 3 Thanos components:

- Thanos-Receive: accepts metrics over http and writes to backend storage.  S3 in our case.
- Thanos-Store-Gateway (or Thanos-Store): exposes a gateway API on top of the backend storage API to retrieve data for queries.
- Thanos-Querier: implements the Prometheus http API to execute queries against the Thanos-Store-Gateway

#### Thanos-Store-Gateway

1. Create a new namespace

   `$ oc create namespace thanos`

1. Fill in `prometheus-federation/manifests/hub/store-s3-secret.yaml`. These will be used by Thano-Receive and Thanos-Store-Gateway to access the backend storage.

   ```
   type: S3
   config:
     bucket: $BUCKET_NAME
     endpoint: s3.amazonaws.com
     region: us-east-1
     access_key: $AWS_ACCESS_KEY_ID
     secret_key: $AWS_SECRET_ACCESS_KEY
     insecure: false
   ```

   With the name of your S3 bucket and AWS keys.

1. Create a secret from the store-s3-secret.yaml file.

   `$ oc --context hub -n thanos create secret generic store-s3-credentials --from-file=./manifests/hub/store-s3-secret.yaml`

1. Thanos-store-gateway requires `anyuid` to work on OCP 4.x, so we'll create a Service Account and attach the anyuid SCC.

   `$ oc --context hub -n thanos create serviceaccount thanos-store-gateway`
   `$ oc --context hub -n thanos adm policy add-scc-to-user anyuid -z thanos-store-gateway`

1. Deploy the thanos-store-gateway

   `$ oc --context hub -n thanos create -f ./manifests/hub/store-gateway.yaml`

   This will create a StatefulSet.  Check that the pods have started:
   `$ oc --context hub -n thanos get pods`

   *Output:*

   ```
   NAME                              READY   STATUS    RESTARTS   AGE
   thanos-store-gateway-0            1/1     Running   0          10s
   ```

#### Thanos-Receive

For security, a bearer token is used to authenticate metric pushes to the thanos-receive endpoint.  We'll create a Service Account and configure it to enable Openshift oauth-proxy.

1. Create the service account

   `$ oc --context hub -n thanos create serviceaccount thanos-receive`

1. Generate a session secret

   `$ oc --context hub -n thanos create secret generic thanos-receive-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)`

1. Annotate the Service Account to enable oauth redirection

   `$ oc --context hub -n thanos annotate serviceaccount thanos-receive serviceaccounts.openshift.io/oauth-redirectreference.thanos-receive='{"kind":"OAuthRedirectReference","apiVersion":"v1","reference":{"kind":"Route","name":"thanos-receive"}}'`

1. Give the Service Account RBAC privileges to delegate authentication

   `$ oc --context hub -n thanos adm policy add-cluster-role-to-user system:auth-delegator -z thanos-receive`

1. Create the Thanos-Receive instance and wait for the pod to deploy

   `$ oc --context hub -n thanos create -f ./manifests/hub/thanos-receive.yaml`

   `$ oc --context hub -n thanos get pods`

   *Output:*

   ```
   NAME                              READY   STATUS    RESTARTS   AGE
   thanos-receive-0                  2/2     Running   0          1s
   thanos-store-gateway-0            1/1     Running   0          31m
   ```

1. Expose the thanos-receive endpoint

   `$ oc --context hub -n thanos create route reencrypt thanos-receive --service=thanos-receive --port=web-proxy --insecure-policy=Redirect`

##### Thanos-Receive Client Tokens

Remote clusters must authenticate with thanos-receive to push metrics.  For this, they'll need an identity in the hub cluster and RBAC permissions to get the `thanos` namespace.

1. Create a ServiceAccount for each client cluster.

   `$ oc --context hub -n thanos create serviceaccount test-cluster-0`

   `$ oc --context hub -n thanos adm policy add-role-to-user view -z test-cluster-0`

   `$ oc --context hub -n thanos create serviceaccount test-cluster-1`

   `$ oc --context hub -n thanos adm policy add-role-to-user view -z test-cluster-1`

#### Thanos-Querier

Thanos Querier is required to execute PromQL queriers against the metric data in our backend storage.  In order to protect the the web facing GUI, we'll again be creating a cluster-identity and bearer token to authenticate users.

1. Create a new session secret

   `$ oc --context hub -n thanos create secret generic thanos-querier-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)`

1. Create a new service account

   `$ oc --context hub -n thanos create serviceaccount thanos-querier`

1. And deploy thanos-querier
   `$ oc --context hub -n thanos create -f ./manifests/hub/thanos-querier-thanos-receive.yaml`

1. Check to see if the thanos-querier pod has deployed
   `$ oc --context hub -n thanos get pods -l "app=thanos-querier"`
   *Output:*

   ```
   NAME                              READY   STATUS    RESTARTS   AGE
   thanos-querier-65b778cd6d-hfbxg   2/2     Running   2          1m
   thanos-receive-0                  2/2     Running   0          26m
   thanos-store-gateway-0            1/1     Running   0          22m
   ```

1. Expose the thanos-querier web gui
   `$ oc --context hub -n thanos create route reencrypt thanos-querier --service=thanos-querier --port=web-proxy --insecure-policy=Redirect`

#### Grafana

Grafana natively supports the thanos-querier as a data source.  As with all our public facing interfaces, we'll start by configuring oauth.

1. Create a session token

   `$ oc --context east2 -n thanos create secret generic grafana-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)`

1. Create a service account and configure it to redirect to oauth
   `$ oc --context hub -n thanos create serviceaccount grafana`

   `$ oc --context hub -n thanos annotate serviceaccount grafana serviceaccounts.openshift.io/oauth-redirectreference.grafana='{"kind":"OAuthRedirectReference","apiVersion":"v1","reference":{"kind":"Route","name":"grafana"}}'`

1. Grafana has a few configurations that need to be encoded into secrets

   `$ oc --context hub -n thanos create secret generic grafana-config --from-file=grafana.ini`

   `$ oc --context hub -n thanos create secret generic grafana-datasources --from-file=prometheus.yaml=prometheus.json`

1. Lastly, create the grafana dashboard config and deploy grafana
   `$ oc create -f manifests/hub/grafana/grafana-dashboards.yaml`
   `$ oc create -f manifests/hub/grafana/grafana.yaml`

1. Check to see if the grafana pod is running
   `$ oc --context hub get pods`
   *Output:*

   ```
   NAME                              READY   STATUS    RESTARTS   AGE
   grafana-5598bf5b8c-fsfck          2/2     Running   0          3m37s
   thanos-querier-65b778cd6d-hfbxg   2/2     Running   2          14d
   thanos-receive-0                  2/2     Running   0          27h
   thanos-store-gateway-0            1/1     Running   0          15d
   ```

---

At this point, we've completed configuring the hub cluster.  It's ready to receive metrics from authenticated client.  We'll move on to configuring both test clusters, but will return to the hub later on.

You can log into the grafana endpoint and inspection the dashboards.  However, there won't be any data there yet until we configure the test clusters.

Get the grafana route and plug it into your web browser

`$ oc --context hub get route grafana`

*Output:*

```
NAME             HOST/PORT                                                PATH   SERVICES         PORT        TERMINATION          WILDCARD
grafana          grafana-thanos.apps.CLUSTER.DOMAIN.com                 grafana          web-proxy   reencrypt/Redirect   None

```



### Configuring the Test Clusters

The "test" clusters will scrape metrics from the Prometheus instance, which deploys by default with OCP, and streams them to the Thanos-Receive endpoint on the hub cluster.  We'll accomplish this by deploying our own instance of Prometheus to act as a reverse-proxy between the default Prometheus and our hub.



> **IMPORTANT: All actions in this section are to be executed on all test clusters.**



#### Deploying the Prometheus Operator

> These commands assume you named your cluster contexts test-cluster-0, test-cluster-1.  Commands are duplicated to reflect they need to be run on both clusters.

1. Create a new namespace

   `$ oc --context test-cluster-0 create ns thanos`

   `$ oc --context test-cluster-1 create ns thanos`

2. Install the Prometheus Operator via the Operator Hub:

   1. Log into the Openshift console of the test cluster using the kubeadmin-password created during cluster deployment.  Username: `kubeadmin`
   1. On the left sidebar, select the Operators drop-down, then OperatorHub.
   1. Type `Prometheus` into the `Filter By Keyword` bar.
   1. Select the `Prometheus Operator`.  If prompted by the community operator warning, click continue.
   1. Click the Install button
   1. In the `Install Operator` panel set the `Installation Mode` to `A specific namespace on the cluster`
   1. Under the `Installed Namespace` menu, select `Thanos`
   1. Leave the Approval Strategy alone.
   1. Click the Install button

2. Wait for the operator to deploy:
   `$ oc --context test-cluster-0 -n thanos get pods`
   `$ oc --context test-cluster-1 -n thanos get pods`

   *Output:*

   ```
   NAME                                   READY   STATUS    RESTARTS   AGE
   prometheus-operator-74d595756b-bmwt7   1/1     Running   1          1m
   ```

2. The cluster-installed Prometheus endpoint requires TLS certs to scrape metrics from.  The certs are stored in the `openshift-monitoring` namespace, so we can just copy them into our `thanos` namespace.  The following command will create a ConfigMap in the `thanos` namespace with the cert bundle.

   `$ oc --context test-cluster-0 -n openshift-monitoring get configmap serving-certs-ca-bundle --export -o yaml | oc --context test-cluster-0 -n thanos apply -f -`

   `$ oc --context test-cluster-1 -n openshift-monitoring get configmap serving-certs-ca-bundle --export -o yaml | oc --context test-cluster-1 -n thanos apply -f -`

2. Our Prometheus instance will also need RBAC permissions to `get` Services in the `openshift-monitoring` namespace in order to discover the scrap-able `/federate` endpoint.  Create the following ClusterRole / ClusterRoleBinding

   `$ oc --context test-cluster-0 create -f ./manifests/cluster-role.yaml`

   `$ oc --context test-cluster-1 create -f ./manifests/cluster-role.yaml`

6. Next, we need to allow Prometheus to send the scraped data to the Thanos-Receive endpoint.  Remember that we created a service account for external clients to send data. We'll be getting the token for this service account and providing it to our Prometheus instance in the test cluster.  The following command combines these operations.

   `$ oc --context test-cluster-0 -n thanos create secret generic metrics-bearer-token --from-literal=metrics_bearer_token=$(oc --context hub -n thanos serviceaccounts get-token test-cluster-0)`

   `$ oc --context test-cluster-1 -n thanos create secret generic metrics-bearer-token --from-literal=metrics_bearer_token=$(oc --context hub -n thanos serviceaccounts get-token test-cluster-1)`

#### Installing Prometheus

In order to protect our public facing Prometheus instances, configure incoming connections to require to have an in-cluster identity, authenticated through an oauth-proxy.

##### Configure Client Tokens

1. Create session tokens on both test clusters.

   `$ oc --context test-cluster-0 -n thanos create secret generic prometheus-k8s-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)`

   `$ oc --context test-cluster-1 -n thanos create secret generic prometheus-k8s-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)`

1. Annotate the `prometheus-k8s` service account (created by the Prometheus Operator) to redirect to the oauth-proxy.
   `$ oc --context test-cluster-0 -n thanos annotate serviceaccount prometheus-k8s serviceaccounts.openshift.io/oauth-redirectreference.prometheus-k8s='{"kind":"OAuthRedirectReference","apiVersion":"v1","reference":{"kind":"Route","name":"federated-prometheus"}}'`

   `$ oc --context test-cluster-1 -n thanos annotate serviceaccount prometheus-k8s serviceaccounts.openshift.io/oauth-redirectreference.prometheus-k8s='{"kind":"OAuthRedirectReference","apiVersion":"v1","reference":{"kind":"Route","name":"federated-prometheus"}}'`

##### Install Prometheus

Before we can run Prometheus, we need to give it the endpoint in the hub cluster (thanos-receive) to stream metrics to.

1. Get the Thanos-Receive Route:

   `$ THANOS_RECEIVE_HOSTNAME=$(oc --context hub -n thanos get route thanos-receive -o jsonpath='{.status.ingress[*].host}')`

1. Replace the `spec.remote_write.url` placeholder:

   `$ sed -i.orig "s/<THANOS_RECEIVE_HOSTNAME>/${THANOS_RECEIVE_HOSTNAME}/g" prometheus-thanos-receive.yaml` 

3. Deploy Prometheus
   `oc --context test-cluster-0 -n thanos create -f prometheus-thanos-receive.yaml`
   `oc --context test-cluster-0 -n thanos create -f service-monitor-test-cluster-0.yaml`

   `oc --context test-cluster-1 -n thanos create -f prometheus-thanos-receive.yaml`
   `oc --context test-cluster-1 -n thanos create -f service-monitor-test-cluster-1.yaml`

4. Lastly, expose the Prometheus endpoints

   `oc --context test-cluster-0 -n thanos create route reencrypt federated-prometheus --service=prometheus-k8s --port=web-proxy --insecure-policy=Redirect`

   `oc --context test-cluster-1 -n thanos create route reencrypt federated-prometheus --service=prometheus-k8s --port=web-proxy --insecure-policy=Redirect`

## Bask in Glorious Data

That's it! Check back in with your Grafana web GUI.  Navigate to the (only) dashboard and you should see a handful of histogram panels.  Feel free to edit, add, or delete panels - the original dashboard configuration is persisted in the cluster-dashboard ConfigMap.  If you want to revert back to the original config, just hard-refresh grafana (`ctrl-shift-r` on Linux/Windows; `⌘-shift-r` on Mac).

