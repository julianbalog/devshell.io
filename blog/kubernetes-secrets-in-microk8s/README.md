Kubernetes Secrets in MicroK8s
==============================

_Deep dive into how secrets are managed and stored in MicroK8s, noting some of the related security concerns_

https://devshell.io/kubernetes-secrets-in-microk8s/

If you are looking for a desktop version of Kubernetes in your everyday development work on your laptop or workstation, you may consider MicroK8s. [MicroK8s](https://microk8s.io/) is a low footprint, minimal Kubernetes distribution (by [Canonical](https://canonical.com/)) for developers, cloud, clusters, Edge, and IoT.

One of the main features of MicroK8s is the built-in high availability, which could make it an attractive option in production deployments. As you may expect, production environments are subject to stricter security requirements. In this article, we'll look at how secrets are stored in MicroK8s and how secure they might be.

Let's start by installing MicroK8s.

Installing MicroK8s
-------------------

Follow the installation steps described [here](https://microk8s.io/), depending on your platform of choice. In this article we'll cover MicroK8s on macOS, Ubuntu, and RHEL/CentOS. Below are the steps for installing MicroK8s on each of these platforms at the time of this writing.

### Installing MicroK8s on macOS

We use [Homebrew](https://brew.sh/) to install MicroK8s. On macOS, MicroK8s is installed using [Multipass](https://github.com/canonical/multipass), a lightweight cross-platform VM manager. Run the following commands and, when prompted to use and configure Multipass, answer yes:

```
brew update
brew install ubuntu/microk8s/microk8s
```

Now, you can install MicroK8s with the following command:

```
microk8s install
```

By default, the MicroK8s installer creates a Multipass VM with 3 vCPUs, 4 GB RAM and 48 GB disk space. You can be more specific, by specifying the number of vCPUs (2), memory (4 GB) and disk space (30 GB), with the following command:

```
microk8s install --cpu 2 --mem 4 --disk 30
```

Since on macOS MicroK8s runs inside a VM, the easiest way to interact with MicroK8s is via the `kubectl` CLI. Let's install it:

```
brew install kubernetes-cli
```

Now, point the local _kubeconfig_ to the MicroK8s cluster:

```
mkdir ~/.kube
microk8s config > ~/.kube/config
```

Make sure you can interact with MicroK8s:

```
kubectl get nodes
```

The command should yield the following (or similar) output:

```
NAME          STATUS   ROLES    AGE    VERSION
microk8s-vm   Ready    <none>   121m   v1.21.0-3+121713cef81e03
```

### Installing MicroK8s on Ubuntu

Make sure `swap` is turned off:

```
sudo swapoff -a
sudo sed -i '/\s*swap\s*/s/^\(.*\)$/# \1/g' /etc/fstab
```

On Ubuntu we use Snap to install MicroK8s. The following command installs version `1.21` of MicroK8s:

```
sudo snap install microk8s --classic --channel=1.21/stable
```

### Installing MicroK8s on RHEL/CentOS

Make sure `swap` is turned off:

```
sudo swapoff -a
sudo sed -i '/\s*swap\s*/s/^\(.*\)$/# \1/g' /etc/fstab
```

On CentOS  7.6 and newer, we need to install the Snap package manager from the EPEL (Extra Packages for Enterprise Linux) repository:

```
sudo yum install epel-release
```

Next, install and configure Snap:

```
sudo yum install -y snapd
sudo systemctl enable --now snapd.socket
sudo ln -s /var/lib/snapd/snap /snap
```

Finally, install MicroK8s, using a stable channel (e.g. `1.21`):

```
sudo snap install microk8s --classic --channel=1.21/stable
```

With MicroK8s installed, let's create some arbitrary secrets.

Working with Kubernetes Secrets
-------------------------------

We'll use the `kubectl` command to create a generic Kubernetes secret (`hello-login`), storing some login credentials (`username` and `password`):

```
kubectl create secret generic hello-login \
  --from-literal="username=admin" \
  --from-literal="password=P@ssw0rd"
```

After the secret has been created, we can retrieve its JSON representation with the following command:

```
kubectl get secret hello-login -o json
```

You'll notice the related `data` element containing the secrets:

```
"data": {
    "password": "UEBzc3cwcmQ=",
    "username": "YWRtaW4="
}
```

Or you can directly retrieve it with:

```
kubectl get secret hello-login -o jsonpath='{ .data }' && echo
```

The credentials are base64-encoded strings and you can decode them with the following commands, revealing the related secrets in clear text:

```
echo "YWRtaW4=" | base64 --decode && echo
echo "UEBzc3cwcmQ=" | base64 --decode && echo
```

So far, so good. As Kubernetes administrators, we are entitled to view and handle secrets at our own discretion. But are these secrets really secure? We know they are stored in MicroK8s' persistent storage. Standard Kubernetes distributions use `etcd` as their storage backend while MicroK8s uses [Dqlite](https://dqlite.io/). `Dqlite` ("Distributed SQLite") is a fault-tolerant implementation of [SQLite](https://www.sqlite.org/), a ligthweight, fast, embedded, and persistent SQL database, written in C. We can, of course, configure MicroK8s to use `etcd`, but the question still remains: are the secrets encrypted? This question is among the very first we'd be asked in the security review of a Kubernetes deployment.

So, let's take a closer look at our secrets in MicroK8s and `Dqlite`.

MicroK8s Secrets and Dqlite
---------------------------

A brief look at the source code of [MicroK8s on Github](https://github.com/ubuntu/microk8s) reveals that the `Dqlite` endpoint is `localhost:19001` and the related database files are in `${SNAP_DATA}/var/kubernetes/backend`. In our case, the `${SNAP_DATA}` directory is `/var/snap/microk8s/current`.

> **Important Note**
>
> On a macOS system, for the next steps, you need MicroK8s terminal access. The following command connects to the related Multipass VM:
>
> ```multipass connect microk8s-vm``` 

Before connecting to the `Dqlite` database, let's define a helper variable pointing to the `Dqlite` database directory:

```
dbdir="/var/snap/microk8s/current/var/kubernetes/backend"
```

We use the `dqlite` command-line utility to connect to the `Dqlite` database:

```
sudo /snap/microk8s/current/bin/dqlite \
  -c ${dbdir}/cluster.crt \
  -k ${dbdir}/cluster.key \
  -s localhost:19001 k8s
```

At the `dqlite>` command-line prompt, query the current tables in the database schema:

```
select name from sqlite_master where type = "table"
```

We get the following output:

```
kine
sqlite_sequence
```

The MicroK8s data it's in the `kine` table. Let's look for anything similar to our `hello-login` secret:

```
select name from kine where name like "%hello%"
```

We get the following output:

```
/registry/secrets/default/hello-login
```

Now, let's query our `hello-login` secret:

```
select name, value from kine where name = "/registry/secrets/default/hello-login"
```

Here's an excerpt from the output:

```
/registry/secrets/default/hello-login|[107 56 115 0 ...]
```

The output hints to a key-value pair, with the key of `/registry/secrets/default/hello-login` and the value pointing to a sequence of ASCII character codes: `107=k`, `56=8`, `115=s`, etc. Let's copy the entire sequence of ASCII codes within the square brackets and assign it to a variable `value` (showing only an excerpt below):

```
value="107 56 115 ..."
```

To get a more readable and user-friendly representation of the related content, run the following command to decode the sequence of ASCII character codes into plain text:

```
echo "$value" | awk '{
  for(i=1; i<=NF; i++)
    if ($i > 31 && $i < 127 || $i == 10)
      printf("%c", $i);
    else
      printf(" ", $i);
  print "";
}'
```

We notice that the output is not encrypted, and we can see our secrets in clear:

```
k8s 
 
 v1  Secret   
  
 hello-login    default" *$079b6736-74ab-43e9-aa08-a465e19fa8a02 8 B         z   s
 kubectl-create  Update  v1"         2 FieldsV1:A
?{"f:data":{".":{},"f:password":{},"f:username":{}},"f:type":{}}  
 password  P@ssw0rd  
 username  admin  Opaque  "
```

While unencrypted secrets could be safe within the security context of the MicroK8s application domain, an attacker could still gain access to the underlying storage and read the data. This is where "encryption at rest" becomes relevant, to ensure the data is encrypted on disk. Let's look at how to encrypt our secrets in MicroK8s.

Encrypting Secrets at Rest in MicroK8s
--------------------------------------

First, we generate a 32-byte random secret to serve as our encryption key:

```
head -c 32 /dev/urandom | base64
```

In our case, the output is:

```
IQrWOCP9g8yQmeCTMdZBrhPVG8WfgEo31B7ueLgFjo8=
```

Next, we create a simple Kubernetes encryption configuration file, named `k8s-crypto.yaml`. You may save this file to a location of your choice. We'll put it in our home folder (e.g., `/home/ubuntu/`). Copy/paste the key generated above into the `secret` attribute:

```
---
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: k8s-crypto
          secret: IQrWOCP9g8yQmeCTMdZBrhPVG8WfgEo31B7ueLgFjo8=
    - identity: {}
```

As you may notice, we're using a reasonably strong encryption method with [AES](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) and [CBC](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Cipher_block_chaining_(CBC)). Before we go into configuring the MicroK8s API server to use encryption at rest, let's find out more about the `kube-apiserver` configuration. Look up the `kubelite` process with the following command:

```
pgrep -an kubelite
```

In our case, the output is (reformatted for better view):

```
/snap/microk8s/2210/kubelite \
  --scheduler-args-file=/var/snap/microk8s/2210/args/kube-scheduler \
  --controller-manager-args-file=/var/snap/microk8s/2210/args/kube-controller-manager \
  --proxy-args-file=/var/snap/microk8s/2210/args/kube-proxy \
  --kubelet-args-file=/var/snap/microk8s/2210/args/kubelet \
  --apiserver-args-file=/var/snap/microk8s/2210/args/kube-apiserver \
  --kubeconfig-file=/var/snap/microk8s/2210/credentials/client.config \
  --start-control-plane=true
```

We can see that the `--apiserver-args-file` option parameter points to `/var/snap/microk8s/2210/args/kube-apiserver`. Edit the `/var/snap/microk8s/2210/args/kube-apiserver` file (e.g., `sudo vi ...`) and add the following line, pointing the encryption provider configuration to our `k8s-crypto.yaml` file:

```
--encryption-provider-config=/home/ubuntu/k8s-crypto.yaml
```

Save the file and restart the MicroK8s `daemon-kubelite` service:

```
sudo systemctl restart snap.microk8s.daemon-kubelite
```

Our old secrets, including `hello-login`, would still be unencrypted, since secrets are only encrypted on write. The following command performs an in-place update of all secrets in MicroK8s, and re-encrypt them according to the encryption provider we just configured:

```
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

Depending on the size of your cluster, the command may take a while, or you may have to run it in smaller batches, targeting individual namespaces. Alternatively, you can delete the `hello-login` secret (`kubectl delete secret hello-login`) and re-create it.

Now, let's retrace the steps described in the [MicroK8s Secrets and Dqlite](#microk8s-secrets-and-dqlite) section and retrieve our `hello-login` secret again. You'll see that, this time the secret is encrypted.

MicroK8s could be a viable solution for a low footprint Kubernetes cluster and, with a bit of tinkering, you can have your secrets encrypted at rest.

Resources
---------

* [Kubernetes - Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
* [Kubernetes - Managing secrets using kubectl](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/)
* [Kubernetes - Encrypting Secret Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
* [MicroK8s on GitHub](https://github.com/ubuntu/microk8s)