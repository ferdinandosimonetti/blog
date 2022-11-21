Title: Using a Microsoft AKS cluster to test Keydb's resilience with Python clients
Author: Ferdinando Simonetti
Tags: Kubernetes, Kubie, AKS, Redis, Keydb, Python 
Category: Kubernetes
Date: 2022-11-21

Today's task: verifying that a Keydb cluster, when deployed via Helm with default values (apart from activating LoadBalancer Service and a custom password), could be resilient to the sudden *death* of any one of its composing Pods, with no client connection's disruption.
Also today's task: learn to interact with Redis/Keydb via Python.

## References

- **Codeberg GIT repo for this project**: [https://codeberg.org/rimmon1971/kind-python-keydb](https://codeberg.org/rimmon1971/kind-python-keydb)
- **Kubie multicontext manager**: [https://github.com/sbstp/kubie](https://github.com/sbstp/kubie)

- **KeyDB Redis replacement**: [https://docs.keydb.dev/](https://docs.keydb.dev/)
- **KeyDB Helm Chart**: [https://artifacthub.io/packages/helm/enapter/keydb](https://artifacthub.io/packages/helm/enapter/keydb)

- **Using Python to interact with Redis**: [https://realpython.com/python-redis/](https://realpython.com/python-redis/)

## Reusing an existing, test, AKS cluster

Here we are:

```
ferdi@DESKTOP-NL6I2OD:~/kind-python-keydb$ az aks list -o table
Name    Location       ResourceGroup             KubernetesVersion    CurrentKubernetesVersion    ProvisioningState    Fqdn
------  -------------  ------------------------  -------------------  --------------------------  -------------------  ------------------------------------------------------------------
keydb   francecentral  DefaultResourceGroup-PAR  1.23.12              1.23.12                     Succeeded            keydb-defaultresourceg-f1fbee-d3388059.hcp.francecentral.azmk8s.io
```

Let's extract the related **kubeconfig** file

```
ferdi@DESKTOP-NL6I2OD:~/kind-python-keydb$ az aks get-credentials --admin --name keydb --resource-group DefaultResourceGroup-PAR --file ~/.kube/keydb.yml
The behavior of this command has been altered by the following extension: aks-preview
Merged "keydb-admin" as current context in /home/ferdi/.kube/keydb.yml
```

And making use of that, via **Kubie** multi-context manager.

```
ferdi@DESKTOP-NL6I2OD:~/kind-python-keydb$ sed -i 's/keydb-admin/keydb/g' /home/ferdi/.kube/keydb.yml
ferdi@DESKTOP-NL6I2OD:~/kind-python-keydb$ kubie ctx keydb
[keydb|default] ferdi@DESKTOP-NL6I2OD:~/kind-python-keydb$ kubectl get nodes
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-21172355-vmss000000   Ready    agent   18d   v1.23.12
aks-nodepool1-21172355-vmss000001   Ready    agent   18d   v1.23.12
aks-nodepool1-21172355-vmss000002   Ready    agent   18d   v1.23.12
```

## Installing Keydb cluster with default values (except for a password)

First of all, we need to add Keydb's Helm repo:

```
[kind|default] ferdi@DESKTOP-NL6I2OD:~/kind-python-keydb$ helm repo add enapter https://enapter.github.io/charts/
"enapter" has been added to your repositories
```

Now we can perform a multi-master, active replica, three node Keydb cluster installation (with our carefully chosen password).
Discovering what does our password mean is left as an exercise for the reader :-)

```
[kind|default] ferdi@DESKTOP-NL6I2OD:~/kind-python-keydb$ helm upgrade --install keydb enapter/keydb --set password="Savignone.2015" --set loadBalancer.enabled=true
Release "keydb" does not exist. Installing it now.
NAME: keydb
LAST DEPLOYED: Mon Nov 21 11:15:06 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

## Reaching it 

We're going to run our Python code from the local PC, so we need to discover the Public IP through which our Keydb cluster is exposed.

```
[keydb|default] ferdi@DESKTOP-NL6I2OD:~/kind-python-keydb$ kubectl get svc
NAME             TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE
keydb            ClusterIP      10.0.137.22   <none>        6379/TCP,9121/TCP   19m
keydb-headless   ClusterIP      None          <none>        6379/TCP            19m
keydb-lb         LoadBalancer   10.0.74.115   EDITED        6379:31052/TCP      19m
kubernetes       ClusterIP      10.0.0.1      <none>        443/TCP             18d
```
