# Pod Security Admission Demo

Create a new project named `psa-test`:

`oc new-project psa-test`

Switch to the newly created project:

`oc project psa-test`

Observe the labels created for this namespace:

`oc get ns psa-test -o jsonpath-as-json='{.metadata.labels}'`

Note the following labels, specifically:

```
    "pod-security.kubernetes.io/audit": "restricted",
    "pod-security.kubernetes.io/audit-version": "v1.24",
    "pod-security.kubernetes.io/warn": "restricted",
    "pod-security.kubernetes.io/warn-version": "v1.24"
```

These are [Pod Security Admission labels](https://kubernetes.io/docs/concepts/security/pod-security-admission/#pod-security-admission-labels-for-namespaces) and are created automatically via [pod security admission synchronization](https://docs.openshift.com/container-platform/4.11/authentication/understanding-and-managing-pod-security-admission.html#security-context-constraints-psa-opting_understanding-and-managing-pod-security-admission) in Openshift 4.11

Create the simple Deployment:

`oc create -f https://raw.githubusercontent.com/radikaled/psa/main/deploy/psa-test-deployment.yaml`

Observe the warning returned:

> Warning: would violate PodSecurity "restricted:v1.24": allowPrivilegeEscalation != false (container "nginx-120" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx-120" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx-120" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx-120" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
> 
> deployment.apps/psa-test created

Despite the warning, the Deployment was created succesfully. Although this behavior will likely change when the `restricted` Pod Security level is enforced.

The `restricted` Pod Security level criteria is outlined [here](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted)

To alleviate any anxiety from the warnings, let's recreate the Deployment with a modified version:

`oc replace -f https://raw.githubusercontent.com/radikaled/psa/main/deploy/psa-test-deployment_restricted.yaml`

Observe that the warnings are now gone.

In order to get rid of the warnings, additional parameters were specified under `securityContext` as outlined in the `restricted` Pod Security level criteria:

```
diff --git a/deploy/psa-test-deployment.yaml b/deploy/psa-test-deployment_restricted.yaml
index 73fa0c0..b4c9612 100644
--- a/deploy/psa-test-deployment.yaml
+++ b/deploy/psa-test-deployment_restricted.yaml
@@ -16,6 +16,12 @@ spec:
       labels:
         app: psa-test
     spec:
+      securityContext:
+        runAsNonRoot: true
+        # Do not use SeccompProfile if your project must work on 
+        # old k8s versions < 1.19 and Openshift < 4.11 
+        seccompProfile:
+          type: RuntimeDefault
       containers:
       - command:
         - nginx
@@ -26,3 +32,8 @@ spec:
         ports:
         - containerPort: 8080
         resources: {}
+        securityContext:
+          allowPrivilegeEscalation: false
+          capabilities:
+            drop: 
+              - ALL
```

So what may things look like once Pod Security Admission is set to enforce? We can illustrate the behavior by attempting to deploy a privileged pod.

Delete any current deployment of `psa-test`:

`oc delete deployment psa-test`

Add the following labels to the `psa-test` namespace:

`oc edit ns psa-test`

```
pod-security.kubernetes.io/enforce: restricted
pod-security.kubernetes.io/enforce-version: v1.24
```

The namespace labels should now resemble the following:

```
  labels:
    kubernetes.io/metadata.name: psa-test
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.24
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.24
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.24
```

Create the `privileged` ServiceAccount:

`oc create -f https://raw.githubusercontent.com/radikaled/psa/main/deploy/sa-privileged.yaml`

Create the `scc-privileged` Role:

`oc create -f https://raw.githubusercontent.com/radikaled/psa/main/deploy/role-scc-privileged.yaml`

Create the `scc-privileged` RoleBinding:

`oc create -f https://raw.githubusercontent.com/radikaled/psa/main/deploy/rb-scc-privileged.yaml`

Finally create the Deployment that utilizes the `privileged` SCC:

`oc create -f https://raw.githubusercontent.com/radikaled/psa/main/deploy/psa-test-deployment_privileged.yaml`

Notice that no warning has been returned via CLI and the Deployment has been created:

`oc get deployments`

Although our Deployment is not READY:

```
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
psa-test   0/1     0            0           2m11s
```

Looking at the events in the namespace reveals the reason:

`oc get events`

> 25s         Warning   FailedCreate        replicaset/psa-test-7cdc48784   (combined from similar events): Error creating: pods "psa-test-7cdc48784-b6wjc" is forbidden: violates PodSecurity "restricted:v1.24": privileged (container "nginx-120" must not set securityContext.privileged=true), allowPrivilegeEscalation != false (container "nginx-120" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx-120" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx-120" must set securityContext.runAsNonRoot=true), runAsUser=0 (container "nginx-120" must not set runAsUser=0), seccompProfile (pod or container "nginx-120" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")

Since the Pod Security Admission level is set to `restricted` and the requisite labels are set to enforce. The pod has been rejected accordingly.