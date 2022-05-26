+++
author = "Robert Radi"
title = "Kubernetes Service Account and their tokens"
date = "2022-01-15"
description = "This article gives an in-depth look  into service accounts in Kubernetes."
tags = [
    "kubernetes",
    "serviceaccount",
    "apiserver",
]
categories = [
    "kubernetes",
    "in-depth",
]
series = ["Themes Guide"]
aliases = ["migrate-from-jekyl"]
+++

If we want to use our Kubernetes cluster we have to talk to the Kubernetes api-server. But who is we? How does the api-server know who is talking to him? If no information provided then the api-server handles the request as an anonymous request. We do not want to allow an anonymous person to do anything in our cluster. That's why we care about authentication in Kubernetes. 

## Basics
There are two types of users is Kubernetes:
* normal users, just like you and me a.k.a human users
* machine users, a.k.a service accounts

Normal users are managed outside of Kubernetes, there are no native Kubernetes objects which represent a human user. However, Kubernetes is able to perform authorization on human users by taking a look - *if a certificate based authentication has been configured* - at the CN field of the client certificate of the user.

Service accounts exist in the context of Kubernetes. These are native namespaced resources and can be created, managed through the api-server. 

## Identity of a POD
We can attach a service account to a POD, and then the POD will be able to send requests to the api-server. What happens if we do not define a service account explicitly? In that case, the service account admission controller will attach the default service account to the POD. 

Wait a minute.. What kind of default service account? As I mentioned, service accounts are tied to a specific namespace. When we create a new namespace a default service account will be created by default as well. Furthermore, a secret will be created.   
![kubernetes-default-sa](/personal-website/images/kubernetes_default_sa.png)

The secret two main things:
* CA cert (this is the CA which signs the certificates in the cluster)
* A JWT token in a base64 encoded form. This token holds the following information about the SA:
```json
{
  "iss": "kubernetes/serviceaccount",
  "kubernetes.io/serviceaccount/namespace": "demo",
  "kubernetes.io/serviceaccount/secret.name": "default-token-gs6c6",
  "kubernetes.io/serviceaccount/service-account.name": "default",
  "kubernetes.io/serviceaccount/service-account.uid": "38149f6d-76a4-47ef-9d29-ca7f32d9cd9d",
  "sub": "system:serviceaccount:demo:default"
}
```

It is important to note that this secret will be mounted to the POD as a volume. You can find this inside of your POD on the following path: `/var/run/secrets/kubernetes.io/serviceaccount/token`

## Interacting with the api-server
First of all, we have to retrieve the address of the api-server. We can achieve this by using one of the following commands:
* `kubectl cluster-info`
* `kubectl config view` 

The api-server is able to intercept REST calls. Lets try to call the **version** endpoint: <br>
`curl -X GET https://<API-SERVER-ADDRESS>:<>PORT/version`. The response will be something like this:
```
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

Do not panic, it is normal. By default, the REST API of the Kubernetes api-server is protected by a certificate, so https is the preferred method. To eliminate this error message, you can use the `-k` option with curl which makes possible to use insecure communication. The other solution is to provide the certificate of the CA by using the `--cacert` option. We have already seen one CA related thing, remember? Yes, you can get this CA cert from the secret of one of the service accounts. 

Lets try again the previous curl command. Depending on how you have configured your api server, but most likely you will get a 401 response code. 
```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```

It is the intended behavior, because we have not done any authentication so the api-server has no clue about our identity. Now, it is time to use the previously acquired token from the secret of the service account. We will use that JWT token as a Bearer token:
`curl -X GET --header "Authorization: Bearer $TOKEN" https://<API-SERVER-ADDRESS>:<port>/version --cacert <ca.crt>`

```json
{
  "major": "1",
  "minor": "20",
  "gitVersion": "v1.20.9",
  "gitCommit": "a5e4de7e277a707bd28d448bd75de58b4f1cdc22",
  "gitTreeState": "clean",
  "buildDate": "2021-11-16T01:09:55Z",
  "goVersion": "go1.15.14",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

If we want to do more complicated things, for example list out the pods in the default namesapce, we will get a 403 response code. This statement is valid only for RBAC enabled clusters. I am using an AKS cluster with RBAC enabled. `curl -X GET --header "Authorization: Bearer $TOKEN" https://<API-SERVER-ADDRESS>:<port>/apis/apps/v1/namespaces/default/pod --cacert <ca.crt>`

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "pods.apps is forbidden: User \"system:serviceaccount:demo:default\" cannot list resource \"pods\" in API group \"apps\" in the namespace \"default\": Azure does not have opinion for this user.",
  "reason": "Forbidden",
  "details": {
    "group": "apps",
    "kind": "pods"
  },
  "code": 403
}
```

By default, service accounts has almost no permissions. You can assign permissions to service accounts by using Roles, ClusterRoles, RoleBindings, ClusterRoleBindings. But this is a completely another topic and I am not going to discuss it in this article. 

## Bonus
If you want to see the REST calls sent by your kubectl CLI, use the *-v 6* flag in your commands.
