# How to use this app

## Goal:  Deploy network policies so that only pods in specific namespace are allowed connection on port 8080

### First deploy  hello_app.yaml

This manifest has important labels

- app: hello-app

### Second: Deploy hello app

```shell

oc create -f hello-app.yaml

```

### Third: 02 expose the deployment and service

```shell

oc expose deploy hello-app

oc expose svc hello-app

```

### Fourth: Deloy another app in the same project

Now deploy another app (test-app) in the same project where you have deployed hello-app

```shell
create -f test-app.yaml

# Test the connection to the hello-app from test-app
oc rsh test-app-6d56b75c9f-n242w curl http://hello-app:8080 | grep installed
## outPut ###
<p>This page is used to test the proper operation of the HTTP server after it has been installed. If you can read this page, it means that the HTTP server installed at this site is working properly.</p>
```

> This should work by default.

### Five: Create another project e.g. frontend and deploy sampleapp.yaml in it

```shell
oc new-project frontend

oc create -f sampleapp.yaml

Try to access the hello-app from this project.

oc rsh sample-app-856679698b-bchbv curl http://hello-app:8080
# -----------   (### OutPut ###)   -----------#
curl: (6) Could not resolve host: hello-app
command terminated with exit code 6
# -----------   [### OutPut ###]   -----------#
# Above is expected, because we need a ip address (or FQDN of the pod which is nothing but 10-217-0-76.hello-app.network-policy.apps-crc.testing) of the pod to access service across the project


oc get pods -o wide -n network-policy
NAME                         READY   STATUS    RESTARTS   AGE     IP            NODE   NOMINATED NODE   READINESS GATES
hello-app-74447f4c8d-k5x4x   1/1     Running   0          13m     10.217.0.76   crc    <none>           <none>
test-app-6d56b75c9f-n242w    1/1     Running   0          7m46s   10.217.0.81   crc    <none>           <none>

oc rsh sample-app-856679698b-bchbv curl http://10.217.0.76:8080 | grep installed

# -----------   [### OutPut ###]   -----------#
<p>This page is used to test the proper operation of the HTTP server after it has been installed. If you can read this page, it means that the HTTP server installed at this site is working properly.</p>
# -----------   [### OutPut ###]   -----------#

```

## Now, lets change this behavior using network policies.

### To create a network policies you need labels on the pod and also label on the namespace

```shell
# always refer namespace where you wish to create network policy, in my case i have added namespace in the yaml file.

oc create -f deny-all.yaml -n network-policy

```

### Try again the connectivity to hello-app

```shell
# below fails as there is now deny all on hello-app
oc rsh sample-app-856679698b-bchbv curl http://10.217.0.76:8080 | grep installed
```

## Finetune the policy

Now, lets fine tune the policy to allow only pods which has label app=sample-app and reside in the namespace which has label kubernetes.io/metadata.name: frontend.

where did I got kubernetes.io/metadata.name: frontend. oc describe namespaces frontend

```shell
oc describe namespaces frontend
Name:         frontend
Labels:       kubernetes.io/metadata.name=frontend
              pod-security.kubernetes.io/audit=restricted
              pod-security.kubernetes.io/audit-version=latest
              pod-security.kubernetes.io/warn=restricted
              pod-security.kubernetes.io/warn-version=latest
Annotations:  openshift.io/description:
              openshift.io/display-name:
              openshift.io/requester: developer
              openshift.io/sa.scc.mcs: s0:c27,c9
              openshift.io/sa.scc.supplemental-groups: 1000720000/10000
              openshift.io/sa.scc.uid-range: 1000720000/10000
Status:       Active
```

now apply the policy and check the connection. It should work.

## Additional Notes:

- The other files e.g. qa_hello_app.yaml is to learn about labels. e.g. spec.selector.matchLabels and spec.template.metadata.labels. you can ignore test_app.yaml file. It is duplicate of qa_hello_app.yaml