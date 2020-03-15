# Deploying and Operating Multiple Disparate Istio Meshes on a Single Kubernetes Cluster

## Introduction

With a rising demand for **HARD** multitenancy in Kubernetes, for individual customers of a SaaS offering or teams within an organization. This demand coupled with the mass adoption of service-meshes (with Istio being the more popular of the choices), we are starting to see a need to support multiple meshes within a single Kubernetes cluster. Additionally, as more independent software vendors (ISVs) like us at [Affirmed Network](https://www.affirmednetworks.com/), move away from delivering standalone applications towards a model of delivering autonomous applications with a dedicated operational and observable infrastructure (like Certmanager, Hashicorp Vault, Prometheus & Grafana, and the EL[F]K stack), there is an ever-increasing need to support disparate service-meshes within a cluster, either the ones operated by the cluster administrator or delivered as a solution by the software vendors, without stepping on each other's managed services.

Current releases of Istio, provide excellent documented support for deploying meshes that span across multiple clusters but fail to provide a clear solution for the inverse scenario, where multiple meshes can operate simultaneously, in isolation, in a single cluster.

---

In this tutorial, I will attempt to provide step-by-step instructions to deploy and operate multiple Istio control planes (or meshes) running concurrently in a single Kubernetes cluster.

After mucking with Istio manifests I eventually arrived at a simple solution that requires no source code changes and can entirely be achieved with deployment manifest changes. The solution enables administrators to operator disparate Istio control planes and have their (injected) `istio-proxy` sidecars associate and communicate with their respective control planes.

### Note on Automatic Sidecar Injection

[__Automatic Sidecar Injection__](https://istio.io/docs/ops/configuration/mesh/injection-concepts/) in Istio leverages [`MutatingWebhookConfigurations`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) to inject the [`istio-proxy`](https://github.com/istio/proxy) sidecar container and the `istio-init` init-container (unless you are using the CNI option) into the selected workload pods. By default, istio requires `istio-injection: enabled` label to be applied to namespaces (when selecting all pods in a namespace) or to specific `deployments` or individual `pods` to enable automatic sidecar injection.

To allow each mesh to inject its sidecar automatically, into the target workload and have them associated with that mesh, we will be modifying the `MutatingWebhookConfiguration` to watch for (non-default) unique/combo labels.

## Instructions

### Modifying the manifests

#### Cluster Scoped Resource

Cluster scoped resources, from Istio manifests, can lead to name collisions if the default names from the upstream helm charts are used. To prevent this from occurring we will need to give each of the [`ClusterRole`](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole), [`ClusterRoleBindings`](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#rolebinding-and-clusterrolebinding) and [`MutatingWebhookConfiguration`]((https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)) & [`ValidatingWebhookConfiguration`]((https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)) resources unique names.

##### ClusterRoles & ClusterRoleBindings

Renaming these `ClusterRole` and `ClusterRoleBinding` is entirely optional since they can be re-used across namespaces if needed. The changes are required if other vendors modify these resource definitions to apply more stringent [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) policies.

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

**IMPORTANT: It is essential to ensure that the resources shown below are uniquely identifiable for every individual control plane deployment.**

- `Webhooks` (and their related `ConfigMap`)

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

Kubernetes allocates unique ports for every `Service` that requests a NodePort. Since two `Service`s cannot use the same [`NodePort`](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport) and we will need to modify the default values for each of the NodePorts associated with the `Gateway` deployments.

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

> CAVEAT: A significant limitation that was uncovered during this effort was that we must statically specify the namespaces that we wish to include in our mesh. To achieve this I have added flag to the citadel container - `- --listened-namespaces=istio-affirmed,istio-affirmed-workspace,kube-system`

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

- Istio [Clean] - [istio-clean.yaml](https://github.com/affirmednetworks/multi-mesh/blob/master/istio-clean.yaml)

```console
helm template install/kubernetes/helm/istio --name istio --namespace istio-affirmed --values install/kubernetes/helm/istio/values-istio-demo.yaml > istio-clean.yaml
```

- Istio [Modified] - [istio-affirmed.yaml](https://github.com/affirmednetworks/multi-mesh/blob/master/istio-affirmed.yaml)

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

- Ensure that the `istio-proxy` sidecar is up and running and is configured to use the right (__istio-affirmed__) control plane services.

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

Citadel is responsible for watching `ServiceAccount` `Secrets` generated by the `API server` as new `ServiceAccount` are created. In return citadel's [`secretcontroller`](https://github.com/istio/istio/blob/1.5.0/security/pkg/k8s/controller/workloadsecret.go) generates new `Secrets` of type `istio.io/key-and-cert` with the `cert-chain.pem`, `key.pem` and `root-cert.pem`. This `Secret` is then used by the `mutating webhook` to authenticate itself with the `API server`. `secretcontroller` leverages [`spiffe`](https://spiffe.io/) to generate a unique identity for every `Service` in the mesh, which is embedded in the `x509` URI of the `cert-chain.pem` file mounted in the `istio-sidecar-injection` pod.

By default, [`citadel`](https://istio.io/docs/ops/deployment/architecture/#citadel) watches `ServiceAccounts` and `Secrets` across all namespaces in the Cluster. However, this may cause contention in a multi control-plane cluster, as one citadel instance from one mesh might step-on, and modify, another mesh's `Secret` and embed an incorrect `spiffe` URI, leading to errors during the certificate authentication between the API Server and the MutatingWebhook endpoint.

`citadel` container takes in command-line arguments that can be used to configure a list of namespaces to be watched by citadel's `secretcontroller`, for events. This, however, is entirely static and requires the administrator to know what namespaces will be part of the service-mesh apriori.

Unfortunately, at this point, there is neither any support for dynamically configuring namespaces to be watched by `citadel` nor a way to provide regex-based command line args to `citadel`.


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

### Istio configuration resources are applied to all istio mesh sidecars

As of istio-1.5.0 there is no support to tag and associated Istio configuration resources, like `VirtualServices`, `Gateways`, `DestinationRules`, etc. to a specific Istio control plane (however, this might change with the introduction of label/annotation-based tagging in the upcoming istio-1.6.0 release).

As a result of this flaw, `istio-proxy` sidecars across multiple meshes will receive envoy `xDS` updates for all deployed `Service`s and Istio configuration resources from every namespace they are deployed to. However, there is a way to limit the amount of configurations a sidecar receives using Istio's [`Sidecar`](https://istio.io/docs/reference/config/networking/sidecar/) resource. All namespaces owned by a single mesh must deploy this additional resource to each of their namespaces to avoid syncing with the other control-planes.

## Closing Remarks

All files shown in the article are available to view and download at [affirmednetworks/multi-mesh](https://github.com/affirmednetworks/multi-mesh).

To learn more about Affirmed Networks' UnityCloud solution for the 5G core click [here](https://www.affirmednetworks.com/resources-draft/solution-brief-affirmed-unitycloud/)
