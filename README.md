# Kube-Platform

Create production ready Development Platform for Kubernetes. It containes tools for:

- Monitoring, Alerting, Logging
- Ingress based adding of DNS entries and TLS Certificates
- Oauth based authentication
- CI/CD tool

In detail - installed tools are:

- Ingress Controller
- Prometheus mit node-exporter, Grafana, Alert Manager, kube-state-metrics etc.
- EFK Stack
- External DNS
- cert-manager
- oauth2-proxy
- keycloak
- argo Workflow, argo-events

## Precondition

- Installed [kustomize 2.0.1](https://github.com/kubernetes-sigs/kustomize/releases)
- Running GKE Kubernetes Cluster with at least ```n1-standard-2``` machines

## Configuration

What you need to know now:

- An Email Adress for HTTPS-Certificate issues
- Your new DNS zone name (e.g. ```kubeplatform.my.domain.io```)
- A GCP project ID (e.g. ```my-google-project-223304```)

### Own OAUTH provider

KubePlatform comes with a preconfigured KeyCloak used for user management and oauth2 authentication. If you plan to use your own oauth provider, collect these parameters:

- An Issuer URL for OpenID Connect
- Client ID and its client secret
- Cookie Secret

Add these paramteres to:

- oauth2-proxy.properties
- patches/oauth2-proxy-patch.yaml

---

## Installation

The installation consists basically of these parts

1. GCE configuration
1. Overlay Configuration
1. Installing yamls on Kubernetes

### GCE configuration

#### GCE Preparation and DNS Configuration

1. Create a new DNS Zone and a ServiceAccount to be used by ```external-dns``` to add hosts to:

```bash
export PROJECT_ID=my-google-project-223304
export DOMAIN=kubeplatform.my.domain.io

gcloud dns managed-zones create "${DOMAIN//./-}" \
    --dns-name "$DOMAIN." \
    --description "Automatically managed zone by kubernetes.io/external-dns"

gcloud iam service-accounts create ${DOMAIN//./-} \
    --display-name "${DOMAIN//./-} service account for external-dns"

gcloud iam service-accounts keys create ./google-credentials.json \
  --iam-account ${DOMAIN//./-}@$PROJECT_ID.iam.gserviceaccount.com

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member serviceAccount:${DOMAIN//./-}@$PROJECT_ID.iam.gserviceaccount.com --role roles/dns.admin
```

2. Ensure that the downloaded credential file `google-credentails.json` is the `google-overlay` folder
3. Make a note of the nameservers that were assigned to your new DNS zone (and enter them as NS entries in your providers DNS):

```bash
gcloud dns record-sets list \
    --zone "${DOMAIN//./-}" \
    --name "$DOMAIN." \
    --type NS
```

### Overlay Configuration

Configuration is made in these three files:

- __kubeplatform.properties__
  - Enter the desired domain (e.g. ```DOMAIN=kubeplatform.my.domain.io```)
  - Enter the GCE project (e.g. ```PROJECT=my-google-project-223304```)
- __cluster-issuer-patch.yaml__
  - Enter two email adresses for Letsencrypt certificate. One for staging and one (or the same) for prod.
- __kustomization.yaml__ 
  - Choose ```namePrefix```, ```nameSuffix``` and ```namespace```
  - If you plan to use letencrypt `prod` environment instead of `staging`, change var `CLUSTER_ISSUER_NAME` accordingly

### Installing yamls

1. create Kubernetes cluster and retrieve kubectl [credentials](https://cloud.google.com/sdk/gcloud/reference/container/clusters/get-credentials)
1. ```kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=my@google.account.com```
1. create namespace you have chosen above
1. execute ```kustomize build google-overlay | kubectl apply -f -```

### Finalize

Wait until your PODs are running

Setup a User in Keycloak:

1. A call to https://keycloak.$(DOMAIN)/auth/admin/ should point you to your Keycloak instance (usename is ```keycloak``` password refer to your kustomization.yaml)
1. Add a user of your choice in Manage/Users (must have an email adress). Please refer to the respective keycloak documentation

You should then be able to use this user to go to:

- https://prometheus.$(DOMAIN)
- https://kibana.$(DOMAIN)
- https://grafana.$(DOMAIN)
- https://argo.$(DOMAIN)

---

## using ARGO

For running basic workflows refer to the [demos](https://github.com/argoproj/argo/blob/master/demo.md) page.

For using it for CI refer to this [example](examples/ci/CI.md)
