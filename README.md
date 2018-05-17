# Dynatrace Kubernetes ActiveGate Plugin

Information: dominik.sachsenhofer@dynatrace.com ([Twitter](https://twitter.com/sachsenhofer))

This is the home of Dynatrace Kubernetes ActiveGate Plugin. This plugin can be used to monitor a Kubernetes Cluster and import metrics from Prometheus endpoints. __It is released as a Developer Preview. It is intended to provide early-stage insights into new features until the Dynatrace Kubernetes Integration/Dashboard becomes available.__

<br>
<br>

## Disclaimer

__The Dynatrace Kubernetes ActiveGate Plugin and the ActiveGate Plugin technology are currently in EAP.__

__Early Access releases provide early-stage insight into new features and functionality of the Dynatrace Platform. They enable you to provide feedback that can significantly impact our direction and implementation.__

__While Early Access releases aren't ready to be used to build production solutions, they're at a stage where you can test and tinker with an implementation. As we receive feedback and iterate on a project, we anticipate breaking changes without advanced warning, so Early Access releases should not be used in a user-facing manner or applied to production environments.__

<br>
<br>


## Overview

The Dynatrace Kubernetes ActiveGate Plugin is a remote based plugin that runs on the Dynatrace ActiveGate.
The plugin systematically requests the Kubernetes API server to get information about nodes, services, deployments and pods on the Kubernetes cluster. In addition, it scrapes Prometheus endpoints to integrate cluster metrics into Dynatrace.

![Img1](./images/img1.png)

<br>
<br>

## 1 Usage

## 1.1 Create Dynatrace Access

__1.1.1 Create a ServiceAccount:__

Create the following resource on your cluster:

```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dynatrace
  namespace: kube-system
EOF
```

<br>

__1.1.1 Create a ClusterRole:__

Create the following resource on your cluster:

```
cat <<EOF | kubectl create -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: dynatrace
rules:
- apiGroups: [""]
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
- apiGroups: [""] 
  resources: ["services/proxy", "pods/proxy"] 
  verbs: ["get"] 
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
EOF
```

If you use a Google KubernetesEngine Cluster and you run into this issue:

```
Error from server (Forbidden): error when creating "STDIN": clusterroles.rbac.authorization.k8s.io "dynatrace" is forbidden: attempt to grant extra privileges: [PolicyRule{Resources:["nodes"], APIGroups:[""], Verbs:["get"]} PolicyRule{Resources:["nodes"], APIGroups:[""], Verbs:["list"]} PolicyRule{Resources:["nodes"], APIGroups:[""], Verbs:["watch"]} PolicyRule{Resources:["services"], APIGroups:[""], Verbs:["get"]} PolicyRule{Resources:["services"], APIGroups:[""], Verbs:["list"]} PolicyRule{Resources:["services"], APIGroups:[""], Verbs:["watch"]} PolicyRule{Resources:["endpoints"], APIGroups:[""], Verbs:["get"]} PolicyRule{Resources:["endpoints"], APIGroups:[""], Verbs:["list"]} PolicyRule{Resources:["endpoints"], APIGroups:[""], Verbs:["watch"]} PolicyRule{Resources:["pods"], APIGroups:[""], Verbs:["get"]} PolicyRule{Resources:["pods"], APIGroups:[""], Verbs:["list"]} PolicyRule{Resources:["pods"], APIGroups:[""], Verbs:["watch"]} PolicyRule{Resources:["configmaps"], APIGroups:[""], Verbs:["get"]} PolicyRule{NonResourceURLs:["/metrics"], Verbs:["get"]}] user=&{dominik.sachsenhofer@gmail.com  [system:authenticated] map[authenticator:[GKE]]} ownerrules=[PolicyRule{Resources:["selfsubjectaccessreviews"], APIGroups:["authorization.k8s.io"], Verbs:["create"]} PolicyRule{NonResourceURLs:["/api" "/api/*" "/apis" "/apis/*" "/healthz" "/swagger-2.0.0.pb-v1" "/swagger.json" "/swaggerapi" "/swaggerapi/*" "/version"], Verbs:["get"]}] ruleResolutionErrors=[]
```

Then you need to do the following steps:

```
# get current google identity
$ gcloud info | grep Account
Account: [myname@example.org]

# grant cluster-admin to your current identity
$ kubectl create clusterrolebinding myname-cluster-admin-binding --clusterrole=cluster-admin --user=myname@example.org
Clusterrolebinding "myname-cluster-admin-binding" created
```

Reference: https://coreos.com/operators/prometheus/docs/latest/troubleshooting.html


<br>

__1.1.2 Create a ClusterRoleBinding:__

Create the following resource on your cluster:

```
cat <<EOF | kubectl create -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: dynatrace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: dynatrace
subjects:
  - kind: ServiceAccount
    name: dynatrace
    namespace: kube-system
EOF
```

<br>

__1.1.3 Get secret name:__

Execute the following command to get the secret name (Tokens):

```
$ kubectl describe serviceaccount dynatrace -n kube-system
Name:                dynatrace
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   dynatrace-token-s4ttd
Tokens:              dynatrace-token-s4ttd
Events:              <none>
```

<br>

__1.1.4 Get token:__

Execute the following command to get the Bearer Token. The Bearer Token is required later when deploying the ActiveGate plugin in Dynatrace UI.

```
$ kubectl describe secret dynatrace-token-s4ttd -n kube-system
Name:         dynatrace-token-s4ttd
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=dynatrace
              kubernetes.io/service-account.uid=919966d2-28f9-11e8-b142-02ea51c0cda0

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1042 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybwV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJteW5hdHJhY2UtdG9rZW4teGp0ODIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpYxUtYWNjb3VudC5uYW1lIjoiZHluYXRyYWNlIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiNzg0ZWUzMDgtMzk3MS0xMWU4LWI0NzYtMxIzN2M4OWFkYzA4Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmR5bmF0cmFjZSJ9.XEsjaIAR2nAKJL-UpRdkzAOwfBzDqX3O9VpMZ1Tq7FPLZ4Fp-cQEAYezT-MYNN-USpPSAF20fjPYxVqI_-u2Ey7fuJsg_dLTISN7znSbPwfRTJxyH2zUOjmNQiM5zP08XV2G8gcn0mNs5ae7SRSeU1JGH9GGdnFQ_y7R5IL4HtnZv_KKT1cCWbwV1bGJNfYlBfyQGnmsHyBrjJMuaNtFpGzQvgekMAoWaDaFCNdHxNgYj5cymjoz1faSkC9RxUmpnR27yFEb_1eZ-u3Csb8yke6o6vSqMW3YY7HxGJAo-BK-utS_fIMs6XOPkq0pHx5TremXB7GyNt6KhGAaXW4t6A
```

Done!

<br>
<br>

## 1.2 Install Dynatrace ActiveGate

__1.2.1 Requirements:__

Requirements:

- Access to your Dynatrace Account

- The Dynatrace Remote Plugin feature flag must be set by your administrator.

- Operating System: Windows

- Memory: at least 2 GB

<br>

__1.2.2 Download:__

In __Dynatrace UI__, go to __Settings - Deploy Dynatrace - Install Dynatrace Security Gateway - Windows - Download securitygateway.exe__

<br>

__1.2.3 Install:__

Execute the file in your Terminal:

```
C:\Users\Administrator> Dynatrace-Security-Gateway-Windows-1.143.76.exe REMOTE_PLUGIN_SHOULD_INSTALL="true"
```

Then follow the steps in the installer.

Done!

<br>
<br>

## 1.3 Deploy Dynatrace ActiveGate Plugin on the ActiveGate

__1.3.1 Upload plugin to ActiveGate Server:__

On your ActiveGate server, upload the __unzipped plugin__ folder to the plugin_deployment directory:

__C:\Program Files\dynatrace\gateway\components\plugin_deployment\k8s_activegateplugin__

<br>

__1.3.2 Restart Dynatrace Remote Plugin Agent:__

On your __ActiveGate server__, go to __Server Manager - Services__, search for the Dynatrace Remote Plugin Agent and restart the service.

Done!

<br>
<br>

## 1.4 Deploy Dynatrace ActiveGate Plugin on Dynatrace

__1.4.1 Upload plugin to Dynatrace:__

In __Dynatrace UI__, go to __Settings - Monitored technologies - Custom plugins - Upload ActiveGate plugin__

Then upload __zipped plugin__ folder to Dynatrace.

<br>

__1.4.2 Configure plugin:__

- Endpoint: Endpoint1 (custom Name of cluster)
- ID: k8s_cluster_1 (custom ID of cluster)
- URL: https://api.k8s.dev.dynatracelabs.com (Link to Kubernetes API-Server)
- Bearer Token: (see above 1.1.4)
  
Done!
<br>
<br>

## 1.5 Install CoreOS-Operator

In order to get useful metrics from Kubernetes, you have to install the Prometheus-Operator (kube-prometheus).

__1.5.1 Install Helm/Tiller:__

First, install Helm/Tiller:

```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
EOF

helm init --service-account tiller
```

__1.5.2 Install Prometheus-Operator:__

Then install the Prometheus-Operator:

```
helm repo add coreos https://s3-eu-west-1.amazonaws.com/coreos-charts/stable/
helm install coreos/prometheus-operator --name prometheus-operator --namespace monitoring
helm install coreos/kube-prometheus --name kube-prometheus --set global.rbacEnable=true --namespace monitoring
```

Done!

<br>

## Screenshots

![Img2](./images/img2.png)
![Img3](./images/img3.png)
![Img4](./images/img4.png)
![Img5](./images/img5.png)
![Img6](./images/img6.png)

<br>

## Limitations

Limitations:

- The Dynatrace ActiveGateway must be installed on a __Windows host__. This is a requirement of the ActiveGate Plugin technology. Linux support is coming soon.

<br>

## Contributing

See [CONTRIBUTING](CONTRIBUTING.md) for details on submitting changes.

<br>

## License

Dynatrace Kubernetes ActiveGate Plugin is under Apache 2.0 license. See [LICENSE](LICENSE) for details.
