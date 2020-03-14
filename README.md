# Deploying and Operating

Operating Multiple Disparate Istio Meshes on a Single Kubernetes Cluster

## Introduction

The rising demand for hard kubernetes multitenancy, either for customers of a SaaS offering or to support disparate internal teams within an organization, coupled with mass adoption of service-meshes (Istio being the more popular of the choices), we are starting to notice a need for supporting multiple meshes within a single Kubernetes cluster. Additionally as more independant software vendors (ISVs), like us at [Affirmed Network](https://www.affirmednetworks.com/), are moving away from delivering standalone applications, and towards a model of delivering, autonomous applications with their own, strongly coupled, management infrastructure (using upstream open source PaaS components for security, observability, recovery, etc.), there is an increasing need to support disparate service-meshes that do not step over the customer's own mesh or the other vendors' meshes.

Current releases of Istio, albeit provide excellent support for deploying meshes across multiple clusters, do not provide a clear solution for the inverse scenario, wherein multiple meshes can operate simultaneous, in isolation, within a single Kubernetes cluster.

In this tutorial, I will attempt to provide step-by-step instructions to deploy and operate multiple Istio controlplanes (or meshes) running concurrently in a single Kubernetes cluster.

After multiple passes of modifying the istio manifest (I think) I have finally found a solution which requires no forks or code changes and can entirely be acheived with some modifications made to the deployment manifest. The solution allows admininistrators to operator disparate Istio controlplanes and have the injected istio sidecars associate, and communicate with and only with their own respective controlplanes.

### A note on Automatic Sidecar Injection

[__Automatic Sidecar Injection__](https://istio.io/docs/ops/configuration/mesh/injection-concepts/) in Istio leverages [`MutatingWebhookConfigurations`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) to inject the [`istio-proxy`](https://github.com/istio/proxy) and `istio-init` containers into the selected workloads. By default, istio requires `istio-injection: enabled` label to be applied to namespaces (to select all pods in a namespace) or individual deployments/pods to allow automatic sidecar injection.

To allow each mesh to inject it's own sidecar automatically, into workloads associated with that mesh, we will modify the Mutating Webhook Controller to watch for unique labels instead of the aforementioned, default labelSelector.

## Instructions

### Modifying the manifests

#### Cluster Scoped Resource

Cluster scoped resources can and will lead to collisions with other Istio deployments if the default names from the upstream helm charts are utilized. To prevent this from happening we will need to give the [`ClusterRole`](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole), [`ClusterRoleBindings`](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#rolebinding-and-clusterrolebinding) and [`MutatingWebhookConfiguration`]((https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)) and [`ValidatingWebhookConfiguration`]((https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)) resources unique names.

##### ClusterRoles & ClusterRoleBindings

Renaming these `ClusterRole` and `ClusterRoleBinding` is entirely optional since they can be used across namespaces. The changes are required if other vendors have modified the clean upstream resource definitions to apply more stringent [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) policies.

```diff
ClusterRole and ClusterRoleBindings
--- cr_crb_clean.yaml   2020-03-13 12:25:02.000000000 -0400
+++ cr_crb_affirmed.yaml    2020-03-13 12:23:55.000000000 -0400
@@ -56,7 +56,7 @@
 apiVersion: rbac.authorization.k8s.io/v1
 kind: ClusterRole
 metadata:
-  name: kiali
+  name: kiali-affirmed
   labels:
     app: kiali
     chart: kiali
@@ -125,7 +125,7 @@
 apiVersion: rbac.authorization.k8s.io/v1
 kind: ClusterRole
 metadata:
-  name: kiali-viewer
+  name: kiali-viewer-affirmed
   labels:
     app: kiali
     chart: kiali
@@ -333,7 +333,7 @@
 kind: ClusterRole
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
-  name: istio-reader
+  name: istio-reader-affirmed
 rules:
   - apiGroups: ['']
     resources: ['nodes', 'pods', 'services', 'endpoints', "replicationcontrollers"]
@@ -377,7 +377,7 @@
 roleRef:
   apiGroup: rbac.authorization.k8s.io
   kind: ClusterRole
-  name: kiali
+  name: kiali-affirmed
 subjects:
 - kind: ServiceAccount
   name: kiali-service-account
@@ -490,13 +490,13 @@
 apiVersion: rbac.authorization.k8s.io/v1
 kind: ClusterRoleBinding
 metadata:
-  name: istio-multi
+  name: istio-multi-affirmed
   labels:
     chart: istio-1.5.0
 roleRef:
   apiGroup: rbac.authorization.k8s.io
   kind: ClusterRole
-  name: istio-reader
+  name: istio-reader-affirmed
 subjects:
 - kind: ServiceAccount
   name: istio-multi
@@ -505,14 +505,16 @@
 apiVersion: rbac.authorization.k8s.io/v1
 kind: ClusterRoleBinding
 metadata:
-  name: istio-reader
+  name: istio-reader-affirmed
   labels:
     chart: istio-1.5.0
 roleRef:
   apiGroup: rbac.authorization.k8s.io
   kind: ClusterRole
-  name: istio-reader
+  name: istio-reader-affirmed
 subjects:
 - kind: ServiceAccount
   name: istio-reader
```

##### Mutating/Validation Webhook Configurations

**IMPORTANT: It is absolutely essential to ensure that the resources shown below are always uniquely identifiable for each control plane deployment.**

- `Webhooks` (and associated `ConfigMaps`)

```diff
--- webhook_clean.yaml  2020-03-13 12:37:56.000000000 -0400
+++ webhook_affirmed.yaml   2020-03-13 12:37:56.000000000 -0400
@@ -1,18 +1,17 @@
-
 ---
 # Source: istio/charts/sidecarInjectorWebhook/templates/mutatingwebhook.yaml

 apiVersion: admissionregistration.k8s.io/v1beta1
 kind: MutatingWebhookConfiguration
 metadata:
-  name: istio-sidecar-injector
+  name: istio-affirmed-sidecar-injector
   labels:
     app: sidecarInjectorWebhook
     chart: sidecarInjectorWebhook
     heritage: Tiller
     release: istio
 webhooks:
-  - name: sidecar-injector.istio.io
+  - name: affirmed-sidecar-injector.istio.io
     clientConfig:
       service:
         name: istio-sidecar-injector
@@ -27,7 +26,7 @@
     failurePolicy: Fail
     namespaceSelector:
       matchLabels:
-        istio-injection: enabled
+        istio-injection: enabled-affirmed

 ---
 # Source: istio/charts/galley/templates/configmap.yaml
@@ -47,7 +46,7 @@
     apiVersion: admissionregistration.k8s.io/v1beta1
     kind: ValidatingWebhookConfiguration
     metadata:
-      name: istio-galley
+      name: istio-affirmed-galley
       labels:
         app: galley
         chart: galley
@@ -55,7 +54,7 @@
         release: istio
         istio: galley
     webhooks:
-      - name: pilot.validation.istio.io
+      - name: affirmed-pilot.validation.istio.io
         clientConfig:
           service:
             name: istio-galley
@@ -121,7 +120,7 @@
         # endpoint is ready.
         failurePolicy: Ignore
         sideEffects: None
-      - name: mixer.validation.istio.io
+      - name: affirmed-mixer.validation.istio.io
         clientConfig:
           service:
             name: istio-galley
@@ -149,12 +148,13 @@
         failurePolicy: Ignore
         sideEffects: None

+---
 # Source: istio/charts/galley/templates/validatingwebhookconfiguration.yaml

 apiVersion: admissionregistration.k8s.io/v1beta1
 kind: ValidatingWebhookConfiguration
 metadata:
-  name: istio-galley
+  name: istio-affirmed-galley
   labels:
     app: galley
     chart: galley
@@ -162,7 +162,7 @@
     release: istio
     istio: galley
 webhooks:
-  - name: pilot.validation.istio.io
+  - name: affirmed-pilot.validation.istio.io
     clientConfig:
       service:
         name: istio-galley
@@ -228,7 +228,7 @@
     # endpoint is ready.
     failurePolicy: Ignore
     sideEffects: None
-  - name: mixer.validation.istio.io
+  - name: affirmed-mixer.validation.istio.io
     clientConfig:
       service:
         name: istio-galley
```

##### Loadbalancer/NodePort port numbers

Kubernetes allocates unique ports to every `Service` that requests a NodePort. Two Services cannot use the same [`NodePort`](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport) and will have the `API Server` deny the request.

```diff
---
# Source: istio/charts/gateways/templates/service.yaml
--- nodeport_svc_clean.yaml 2020-03-13 12:47:02.000000000 -0400
+++ nodeport_svc_affirmed.yaml  2020-03-13 12:47:03.000000000 -0400
@@ -4,20 +4,20 @@
 apiVersion: v1
 kind: Service
 metadata:
-  name: istio-ingressgateway
+  name: istio-affirmed-ingressgateway
   namespace: istio-affirmed
   annotations:
   labels:
     chart: gateways
     heritage: Tiller
     release: istio
-    app: istio-ingressgateway
+    app: istio-affirmed-ingressgateway
     istio: ingressgateway
 spec:
   type: LoadBalancer
   selector:
     release: istio
-    app: istio-ingressgateway
+    app: istio-affirmed-ingressgateway
     istio: ingressgateway
   ports:
```

##### Deployments - Container command and arguments

> CAVEAT: A significant limitation that was uncovered in this chart modification based approach is that we must know apriori the namespaces that we want to be governed by our own Istio deployment.

```diff
--- deployment_clean.yaml   2020-03-13 13:10:34.000000000 -0400
+++ deployment_affirmed.yaml    2020-03-13 13:10:28.000000000 -0400
# Source: istio/charts/galley/templates/deployment.yaml
@@ -53,6 +53,7 @@
           - --enable-reconcileWebhookConfiguration=true
           - --monitoringPort=15014
           - --log_output_level=default:info
+          - --webhook-name=istio-affirmed-galley
           volumeMounts:
           - name: certs
             mountPath: /etc/certs
# Source: istio/charts/security/templates/deployment.yaml
@@ -177,6 +178,7 @@
             - --monitoring-port=15014
             - --self-signed-ca=true
             - --workload-cert-ttl=2160h
+            - --listened-namespaces=istio-affirmed,istio-affirmed-workspace,kube-system
           env:
             - name: CITADEL_ENABLE_NAMESPACES_BY_DEFAULT
               value: "true"
# Source: istio/charts/sidecarInjectorWebhook/templates/deployment.yaml
@@ -265,6 +267,8 @@
             - --healthCheckInterval=2s
             - --healthCheckFile=/tmp/health
             - --reconcileWebhookConfig=true
+            - --webhookName=affirmed-sidecar-injector.istio.io
+            - --webhookConfigName=istio-affirmed-sidecar-injector
           volumeMounts:
           - name: config-volume
             mountPath: /etc/istio/config
```

### Generate, deploy and verify

The steps shown below can be replicated to any number of control plane by means of substitution. Try it out with an additional control plane.

#### Requirements

- A Kubernetes Cluster
- [Helm](https://helm.sh/)
- [Kubectl](https://kubernetes.io/docs/reference/kubectl/overview/)

#### Installation

Download release istio-1.5.0 (installation details can be found [here](https://istio.io/docs/setup/getting-started/#download))

Go to the Istio release page to download the installation file for your OS, or download and extract the latest release automatically (Linux or macOS):

```console
curl -L https://istio.io/downloadIstio | sh -
```

Move to the Istio package directory. For example, if the package is istio-1.5.0:

```console
cd istio-1.5.0
```

#### Generate the manifests

- Add the istio helm repo

```console
helm repo add istio.io https://storage.googleapis.com/istio-release/releases/1.5.0/charts/
```

- Istio [Clean] - istio-clean.yaml

```console
helm template install/kubernetes/helm/istio --name istio --namespace istio-affirmed --values install/kubernetes/helm/istio/values-istio-demo.yaml > istio-clean.yaml
```

- Istio [Modified] - istio-affirmed.yaml

```console
# Apply the patch file to the clean istio yaml and also backup the original
patch -b istio-clean.yaml istio.patch

# Rename istio-clean.yaml to the new modified version istio-affirmed.yaml
mv istio-clean.yaml istio-affirmed.yaml

# Rename the backup generated by the patch function to istio-clean.yaml
mv istio-clean.yaml.orig istio-clean.yaml
 Kubectl apply the generated manifests to your Kubernetes Cluster
```

#### Deploy the control plane

- Create the mesh control plane Namespace

```console
kubectl create namespace istio-affirmed
```

- Deploy all CRDs to the namespace

```console
helm template install/kubernetes/helm/istio-init --name istio-init --namespace istio-affirmed | kubectl apply -f -
```

- Apply manifest to the istio namespace

```console
kubectl create -f istio-affirmed.yaml
```

#### Verify sidecar injection works with a sample NGINX deployment (See nginx-affirmed.yaml)

- Create a workload namespace and label it to enable sidecar injection

```console
kubectl create namespace nginx-affirmed
kubectl label namespace nginx-affirmed istio-injection=enabled-affirmed
```

- Deploy NGINX application to labeled namespace

```console
kubectl create -f nginx-affirmed.yaml
```

- Verify the deployment

```console
kubectl get pods -n nginx-affirmed
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-54f57cf6bf-h9n9t   2/2     Running   0          10s
```

- Ensure that the istio-proxy sidecar is deployed and that it is pointed to the istio-affirmed control-plane

```console
kubectl describe pod nginx-deployment-54f57cf6bf-h9n9t
```

```console
# file truncated
Name:         nginx-deployment-54f57cf6bf-h9n9t
Namespace:    nginx-affirmed
...
Init Containers:
  istio-init:
  ...
Containers:
  nginx:
  ...
  istio-proxy:
    Args:
      proxy
      sidecar
      --domain
      $(POD_NAMESPACE).svc.cluster.local
      ...
      # Verification checkpoint. MUST point to deployed istio control-plane
      # This acknowledges that the mutatingwebhook did the right thing.
      --discoveryAddress
      istio-pilot.istio-affirmed:15010
      ...
      --zipkinAddress
      zipkin.istio-affirmed:9411
      ...
```

## Caveats and Gotchas

### Owned/managed namespaces must be specified in advance

Citadel is responsible for watching `ServiceAccount` `Secrets` generated by the `API server` as new `ServiceAccount` are created. In return citadel's [`secretcontroller`](https://github.com/istio/istio/blob/1.5.0/security/pkg/k8s/controller/workloadsecret.go) generates new `Secrets` of type `istio.io/key-and-cert` with the `cert-chain.pem`, `key.pem` and `root-ca.pem`. This `Secret` is then used by the `mutating webhook` to authenticate itself with the `API server`. SecretController leverages [`spiffe`](https://spiffe.io/) to generate identities of the services and embeds the `spiffe` URI in the certificate resource mounted by the sidecar injector pod.

By default, [`citadel`](https://istio.io/docs/ops/deployment/architecture/#citadel) watches these `ServiceAccounts` and `Secrets` across all namespaces in the Cluster. However this causes contention in a multi control-plane scenario as one citadel instance from Mesh-A might step-on and modify a Mesh-B secret, leading to Identity errors during `x509` authentication.

The `citadel` binary provides flags to override the default and allows us to specify one or more namespaces that citadel's secretcontroller must watch for events. This, however, is entirely static in nature and requires the administrator to know what namespaces will be part of the service-mesh apriori.

Unfortunately, at this point, there is no support to dynamically specify the namespaces or provide regex-based watchers.

### Istio configuration resources are applied to all istio mesh sidecars

As of istio-1.5.0 there is no support to tag Istio configuration resources, like VirtualServices, Gateways, DestinationRules, etc. to a unique Istio control plane (this might change with the introduction of tagging in the upcoming istio-1.6.0 release).

Due to this fact all disparate mesh sidecars will recieve envoy `xDS` updates for all services and resources in the entire namespace. However, there is a way to limit the number of configurations a sidecar receives using Istio Sidecar resource (or Envoy Filter). All namespaces owned by a single mesh must deploy this additional resource to their namespaces to avoid syncing with the other control-planes.
Notice, below, the addition parameter `--listened-namespaces` passed to the `citadel` container

```diff

# Source: istio/charts/security/templates/deployment.yaml

@@ -177,6 +178,7 @@
             - --monitoring-port=15014
             - --self-signed-ca=true
             - --workload-cert-ttl=2160h
+            - --listened-namespaces=istio-affirmed,istio-affirmed-workspace,kube-system
           env:
             - name: CITADEL_ENABLE_NAMESPACES_BY_DEFAULT
               value: "true"
```
