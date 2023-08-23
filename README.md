# HOWTO install linkerd in openshift OCP 4.10

## 0) Install tools
1) helm
``` bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

2) linkerd
``` bash
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh

$ linkerd version
Client version: stable-2.13.6
Server version: stable-2.13.6
```
3) OCP version
``` bash
[lab-user@bastion ~]$ oc version
Client Version: 4.10.36
Server Version: 4.10.36
Kubernetes Version: v1.23.5+8471591

```


## 1) Install Linkerd CNI
This is a Network Plugin (Container Network Interface), in this case it will install linkerd binary and other stuff
on openshift nodes.

``` bash
1) oc new-project linkerd-cni

[lab-user@bastion ~]$ oc new-project linkerd-cni
Now using project "linkerd-cni" on server "https://api.cluster-47pjr.47pjr.sandbox1131.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app rails-postgresql-example

to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=k8s.gcr.io/e2e-test-images/agnhost:2.33 -- /agnhost serve-hostname

2) oc annotate ns linkerd-cni linkerd.io/inject=disabled

[lab-user@bastion ~]$ oc annotate ns linkerd-cni linkerd.io/inject=disabled
namespace/linkerd-cni annotated

3) oc adm policy add-scc-to-user privileged -z linkerd-cni -n linkerd-cni

[lab-user@bastion ~]$ oc adm policy add-scc-to-user privileged -z linkerd-cni -n linkerd-cni
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "linkerd-cni"

4) Helm repo add
helm repo add linkerd https://helm.linkerd.io/stable

5) Helm install
[lab-user@bastion ~]$ helm install linkerd2-cni \
 --set installNamespace=false \
 --set destCNIBinDir=/var/lib/cni/bin \
 --set destCNINetDir=/etc/kubernetes/cni/net.d \
 linkerd/linkerd2-cni
NAME: linkerd2-cni
LAST DEPLOYED: Sat Aug 19 15:02:28 2023
NAMESPACE: linkerd-cni
STATUS: deployed
REVISION: 1
TEST SUITE: None

6) Edit the DaemonSet and add:  privileged= true

7) Check that DS starts ok
[lab-user@bastion ~]$ oc get pods
NAME                READY   STATUS    RESTARTS   AGE
linkerd-cni-26xd5   1/1     Running   0          94s
linkerd-cni-7tmqk   1/1     Running   0          95s
linkerd-cni-bqv4k   1/1     Running   0          94s
linkerd-cni-ddvgx   1/1     Running   0          95s
linkerd-cni-hrqlx   1/1     Running   0          94s
linkerd-cni-w879f   1/1     Running   0          94s

```

## 2) Install Linkerd Control Plane
This wil have a pod to inject proxy, get traces, an api, identity,service discovery.

``` bash
1) oc new-project linkerd

[lab-user@bastion ~]$ oc new-project linkerd
Now using project "linkerd" on server "https://api.cluster-47pjr.47pjr.sandbox1131.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app rails-postgresql-example

to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=k8s.gcr.io/e2e-test-images/agnhost:2.33 -- /agnhost serve-hostname

2) oc annotate ns linkerd linkerd.io/inject=disabled

[lab-user@bastion ~]$ oc annotate ns linkerd linkerd.io/inject=disabled
namespace/linkerd annotated
[lab-user@bastion ~]$ oc label ns linkerd linkerd.io/control-plane-ns=linkerd \
   linkerd.io/is-control-plane=true \
   config.linkerd.io/admission-webhooks=disabled
namespace/linkerd labeled

3)
oc adm policy add-scc-to-user privileged -z default -n linkerd
oc adm policy add-scc-to-user privileged -z linkerd-destination -n linkerd
oc adm policy add-scc-to-user privileged -z linkerd-identity -n linkerd
oc adm policy add-scc-to-user privileged -z linkerd-proxy-injector -n linkerd
oc adm policy add-scc-to-user privileged -z linkerd-heartbeat -n linkerd

[lab-user@bastion ~]$ oc adm policy add-scc-to-user privileged -z default -n linkerd
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "default"
[lab-user@bastion ~]$ oc adm policy add-scc-to-user privileged -z linkerd-destination -n linkerd
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "linkerd-destination"
[lab-user@bastion ~]$ oc adm policy add-scc-to-user privileged -z linkerd-identity -n linkerd
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "linkerd-identity"
[lab-user@bastion ~]$ oc adm policy add-scc-to-user privileged -z linkerd-proxy-injector -n linkerd
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "linkerd-proxy-injector"
[lab-user@bastion ~]$ oc adm policy add-scc-to-user privileged -z linkerd-heartbeat -n linkerd
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "linkerd-heartbeat"

4)
[lab-user@bastion ~]$ linkerd install --crds | kubectl apply -f -
Rendering Linkerd CRDs...
Next, run `linkerd install | kubectl apply -f -` to install the control plane.

customresourcedefinition.apiextensions.k8s.io/authorizationpolicies.policy.linkerd.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.policy.linkerd.io created
customresourcedefinition.apiextensions.k8s.io/meshtlsauthentications.policy.linkerd.io created
customresourcedefinition.apiextensions.k8s.io/networkauthentications.policy.linkerd.io created
customresourcedefinition.apiextensions.k8s.io/serverauthorizations.policy.linkerd.io created
customresourcedefinition.apiextensions.k8s.io/servers.policy.linkerd.io created
customresourcedefinition.apiextensions.k8s.io/serviceprofiles.linkerd.io created

5) linkerd install --linkerd-cni-enabled --set installNamespace=false | oc apply -f -

[lab-user@bastion ~]$ linkerd install \
>   --linkerd-cni-enabled \
>   --set installNamespace=false |
>   oc apply -f -

Warning: resource namespaces/linkerd is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.
namespace/linkerd configured
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-identity created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-identity created
serviceaccount/linkerd-identity created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-destination created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-destination created
serviceaccount/linkerd-destination created
secret/linkerd-sp-validator-k8s-tls created
validatingwebhookconfiguration.admissionregistration.k8s.io/linkerd-sp-validator-webhook-config created
secret/linkerd-policy-validator-k8s-tls created
validatingwebhookconfiguration.admissionregistration.k8s.io/linkerd-policy-validator-webhook-config created
clusterrole.rbac.authorization.k8s.io/linkerd-policy created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-destination-policy created
role.rbac.authorization.k8s.io/linkerd-heartbeat created
rolebinding.rbac.authorization.k8s.io/linkerd-heartbeat created
clusterrole.rbac.authorization.k8s.io/linkerd-heartbeat created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-heartbeat created
serviceaccount/linkerd-heartbeat created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-proxy-injector created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-proxy-injector created
serviceaccount/linkerd-proxy-injector created
secret/linkerd-proxy-injector-k8s-tls created
mutatingwebhookconfiguration.admissionregistration.k8s.io/linkerd-proxy-injector-webhook-config created
configmap/linkerd-config created
role.rbac.authorization.k8s.io/ext-namespace-metadata-linkerd-config created
secret/linkerd-identity-issuer created
configmap/linkerd-identity-trust-roots created
service/linkerd-identity created
service/linkerd-identity-headless created
deployment.apps/linkerd-identity created
service/linkerd-dst created
service/linkerd-dst-headless created
service/linkerd-sp-validator created
service/linkerd-policy created
service/linkerd-policy-validator created
deployment.apps/linkerd-destination created
cronjob.batch/linkerd-heartbeat created
deployment.apps/linkerd-proxy-injector created
service/linkerd-proxy-injector created
secret/linkerd-config-overrides created
[lab-user@bastion ~]$

6) linkerd check

[lab-user@bastion ~]$ linkerd check
kubernetes-api
--------------
√ can initialize the client
√ can query the Kubernetes API

kubernetes-version
------------------
√ is running the minimum Kubernetes API version

linkerd-existence
-----------------
√ 'linkerd-config' config map exists
√ heartbeat ServiceAccount exist
√ control plane replica sets are ready
√ no unschedulable pods
√ control plane pods are ready
√ cluster networks contains all pods
√ cluster networks contains all services

linkerd-config
--------------
√ control plane Namespace exists
√ control plane ClusterRoles exist
√ control plane ClusterRoleBindings exist
√ control plane ServiceAccounts exist
√ control plane CustomResourceDefinitions exist
√ control plane MutatingWebhookConfigurations exist
√ control plane ValidatingWebhookConfigurations exist
√ proxy-init container runs as root user if docker container runtime is used

linkerd-identity
----------------
√ certificate config is valid
√ trust anchors are using supported crypto algorithm
√ trust anchors are within their validity period
√ trust anchors are valid for at least 60 days
√ issuer cert is using supported crypto algorithm
√ issuer cert is within its validity period
√ issuer cert is valid for at least 60 days
√ issuer cert is issued by the trust anchor

linkerd-webhooks-and-apisvc-tls
-------------------------------
√ proxy-injector webhook has valid cert
√ proxy-injector cert is valid for at least 60 days
√ sp-validator webhook has valid cert
√ sp-validator cert is valid for at least 60 days
√ policy-validator webhook has valid cert
√ policy-validator cert is valid for at least 60 days

linkerd-version
---------------
√ can determine the latest version
√ cli is up-to-date

control-plane-version
---------------------
√ can retrieve the control plane version
√ control plane is up-to-date
√ control plane and cli versions match

linkerd-control-plane-proxy
---------------------------
√ control plane proxies are healthy
√ control plane proxies are up-to-date
√ control plane proxies and cli versions match

Status check results are √
[lab-user@bastion ~]$
```

## 3) Install Observability
This is an extension that contains observability and visualization components for Linkerd.
``` bash

1)oc new-project linkerd-viz

[lab-user@bastion ~]$ oc new-project linkerd-viz
Now using project "linkerd-viz" on server "https://api.cluster-47pjr.47pjr.sandbox1131.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app rails-postgresql-example

to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=k8s.gcr.io/e2e-test-images/agnhost:2.33 -- /agnhost serve-hostname

2) oc annotate ns linkerd-viz linkerd.io/inject=enabled config.linkerd.io/proxy-await=enabled
   oc annotate ns linkerd-viz  linkerd.io/inject=enabled

   config.linkerd.io/proxy-await=enabled
namespace/linkerd-viz annotated
[lab-user@bastion ~]$ oc label ns linkerd-viz \
   linkerd.io/extension=viz
namespace/linkerd-viz labeled

3)
oc adm policy add-scc-to-user privileged -z default -n linkerd-viz
oc adm policy add-scc-to-user privileged -z grafana -n linkerd-viz
oc adm policy add-scc-to-user privileged -z metrics-api -n linkerd-viz
oc adm policy add-scc-to-user privileged -z prometheus -n linkerd-viz
oc adm policy add-scc-to-user privileged -z tap -n linkerd-viz
oc adm policy add-scc-to-user privileged -z web -n linkerd-viz
oc adm policy add-scc-to-user privileged -z tap-injector -n linkerd-viz


[lab-user@bastion ~]$ oc adm policy add-scc-to-user privileged -z default -n linkerd-viz
eus -n linkerd-viz
oc adm policy add-scc-to-user privileged -z tap -n linkerd-viz
oc adm policy add-scc-to-user privileged -z web -n linkerd-viz
oc adm policy add-scc-to-user privileged -z tap-injector -n linkerd-viz

[lab-user@bastion ~]$ oc adm policy add-scc-to-user privileged -z grafana -n linkerd-viz
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "grafana"
[lab-user@bastion ~]$ oc adm policy add-scc-to-user privileged -z metrics-api -n linkerd-viz
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "metrics-api"
[lab-user@bastion ~]$ oc adm policy add-scc-to-user privileged -z prometheus -n linkerd-viz
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "prometheus"
[lab-user@bastion ~]$ oc adm policy add-scc-to-user privileged -z tap -n linkerd-viz
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "tap"
[lab-user@bastion ~]$ oc adm policy add-scc-to-user privileged -z web -n linkerd-viz
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "web"
[lab-user@bastion ~]$ oc adm policy add-scc-to-user privileged -z tap-injector -n linkerd-viz
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "tap-injector"

4) linkerd viz install --set installNamespace=false | oc apply -f -

[lab-user@bastion ~]$ linkerd viz install \
>   --set installNamespace=false |
>   oc apply -f -
Warning: resource namespaces/linkerd-viz is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by oc apply. oc apply should only be used on resources created declaratively by either oc create --save-config or oc apply. The missing annotation will be patched automatically.
namespace/linkerd-viz configured
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-viz-metrics-api created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-viz-metrics-api created
serviceaccount/metrics-api created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-viz-prometheus created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-viz-prometheus created
serviceaccount/prometheus created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-viz-tap created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-viz-tap-admin created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-viz-tap created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-viz-tap-auth-delegator created
serviceaccount/tap created
rolebinding.rbac.authorization.k8s.io/linkerd-linkerd-viz-tap-auth-reader created
secret/tap-k8s-tls created
apiservice.apiregistration.k8s.io/v1alpha1.tap.linkerd.io created
role.rbac.authorization.k8s.io/web created
rolebinding.rbac.authorization.k8s.io/web created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-viz-web-check created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-viz-web-check created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-viz-web-admin created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-viz-web-api created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-viz-web-api created
serviceaccount/web created
service/metrics-api created
deployment.apps/metrics-api created
server.policy.linkerd.io/metrics-api created
authorizationpolicy.policy.linkerd.io/metrics-api created
meshtlsauthentication.policy.linkerd.io/metrics-api-web created
networkauthentication.policy.linkerd.io/kubelet created
configmap/prometheus-config created
service/prometheus created
deployment.apps/prometheus created
server.policy.linkerd.io/prometheus-admin created
authorizationpolicy.policy.linkerd.io/prometheus-admin created
service/tap created
deployment.apps/tap created
server.policy.linkerd.io/tap-api created
authorizationpolicy.policy.linkerd.io/tap created
clusterrole.rbac.authorization.k8s.io/linkerd-tap-injector created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-tap-injector created

5) Add allowPrivilegeEscalation: true with failed inits. (not need it)

6) linkerd viz check
[lab-user@bastion ~]$ linkerd viz check
linkerd-viz
-----------
√ linkerd-viz Namespace exists
√ can initialize the client
√ linkerd-viz ClusterRoles exist
√ linkerd-viz ClusterRoleBindings exist
√ tap API server has valid cert
√ tap API server cert is valid for at least 60 days
√ tap API service is running
√ linkerd-viz pods are injected
√ viz extension pods are running
√ viz extension proxies are healthy
√ viz extension proxies are up-to-date
√ viz extension proxies and cli versions match
√ prometheus is installed and configured correctly
√ viz extension self-check

Status check results are √
```

## 4) Fix dashboard

1) Create a route on linkerd-viz project, web service http (8084)

2) On web deployment add -enforced-host=.* like this:
``` bash
          args:
            - >-
              -linkerd-metrics-api-addr=metrics-api.linkerd-viz.svc.cluster.local:8085
            - '-cluster-domain=cluster.local'
            - '-controller-namespace=linkerd'
            - '-log-level=info'
            - '-log-format=plain'
            - '-enforced-host=.*'
            - '-enable-pprof=false'

```


## 5) Install App to test

``` bash
1)  oc new-project emojivoto

[lab-user@bastion ~]$ oc new-project emojivoto
Now using project "emojivoto" on server "https://api.cluster-47pjr.47pjr.sandbox1131.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app rails-postgresql-example

to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=k8s.gcr.io/e2e-test-images/agnhost:2.33 -- /agnhost serve-hostname

2) oc annotate ns emojivoto linkerd.io/inject=enabled

[lab-user@bastion ~]$ oc annotate ns emojivoto \
   linkerd.io/inject=enabled

namespace/emojivoto annotated

3) Add policy
oc adm policy add-scc-to-user privileged -z default -n emojivoto
oc adm policy add-scc-to-user privileged -z emoji -n emojivoto
oc adm policy add-scc-to-user privileged -z voting -n emojivoto
oc adm policy add-scc-to-user privileged -z web -n emojivoto

[lab-user@bastion ~]$ oc adm policy add-scc-to-user privileged -z default -n emojivoto
 apply -f https://run.linkerd.io/emojivoto.ymlclusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "default"
[lab-user@bastion ~]$ oc adm policy add-scc-to-user privileged -z emoji -n emojivoto
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "emoji"
[lab-user@bastion ~]$ oc adm policy add-scc-to-user privileged -z voting -n emojivoto
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "voting"
[lab-user@bastion ~]$ oc adm policy add-scc-to-user privileged -z web -n emojivoto
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "web"

3) oc apply -f https://run.linkerd.io/emojivoto.yml

[lab-user@bastion ~]$ oc apply -f https://run.linkerd.io/emojivoto.yml
Warning: resource namespaces/emojivoto is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by oc apply. oc apply should only be used on resources created declaratively by either oc create --save-config or oc apply. The missing annotation will be patched automatically.
namespace/emojivoto configured
serviceaccount/emoji created
serviceaccount/voting created
serviceaccount/web created
service/emoji-svc created
service/voting-svc created
service/web-svc created
deployment.apps/emoji created
deployment.apps/vote-bot created
deployment.apps/voting created
deployment.apps/web created

```

## Others:

With this we can fix pending pods:
``` bash
            privileged: true
            runAsUser: 65534
            runAsNonRoot: true
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: true
```
