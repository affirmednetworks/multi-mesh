--- webhook_clean.yaml	2020-03-13 12:37:56.000000000 -0400
+++ webhook_affirmed.yaml	2020-03-13 12:37:56.000000000 -0400
@@ -1,18 +1,17 @@
-
 ---
 # Source: istio/charts/sidecarInjectorWebhook/templates/mutatingwebhook.yaml
 
 apiVersion: admissionregistration.k8s.io/v1beta1
 kind: MutatingWebhookConfiguration
 metadata:
-  name: istio-sidecar-injector
+  name: <your-namespace>-sidecar-injector
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
+      name: <your-namespace>-galley
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
+  name: <your-namespace>-galley
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
