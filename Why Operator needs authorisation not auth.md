## Authentication — Already Handled by Kubernetes

When your operator runs inside the cluster, it runs as a Pod.
That Pod has:


ServiceAccount


Example:

apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitops-operator


Kubernetes automatically:

Mounts a ServiceAccount token inside the Pod

Uses that token when your controller talks to the API server

So when your operator does:

r.Client.Get(...)


It automatically sends:

Bearer <service-account-token>


to the Kubernetes API server.

The API server authenticates it.

So you don’t write authentication logic.

Kubernetes already did.
