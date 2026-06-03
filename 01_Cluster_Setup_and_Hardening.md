---
tags:
  - cks
  - kubernetes
---
## 1. CIS Benchmarks
- CIS Benchmarks are security hardening standards from the Center for Internet Security. 
- Think of them as practical checklists that tell you how to securely configure servers, cloud accounts, Kubernetes, Docker, databases, operating systems, and network devices.
- CIS describes them as prescriptive configuration recommendations for many technology families

> [!example] For a Linux server, CIS Benchmark may say:
> - Disable root SSH login
> - Use strong password policy
> - Enable audit logs
> - Disable unused services
> - Configure firewall
> - Set correct file permissions

> [!example] For Kubernetes, it may say:
> - Disable anonymous access to API Server
> - Enable audit logging
> - Restrict kubelet permissions
> - Use RBAC
> - Do not run containers as root
> - Use network policies
> - Protect etcd

- For Kubernetes, the CIS Kubernetes Benchmark is a hardening checklist for securing the cluster configuration: API Server, etcd, scheduler, controller manager, kubelet, RBAC, network policies, pod security, audit logging, and worker nodes. CIS publishes the Kubernetes Benchmark as secure configuration guidance for Kubernetes clusters.

### kube-bench Tool
- The kube-bench tool is an open source tool from Aqua Security that can perform automated assessments to check whether Kubernetes is deployed as per security best practices.
- checks whether Kubernetes is deployed securely by running tests based on the CIS Kubernetes Benchmark. It supports running inside the cluster as a [[#Job|^Kubernetes Job]] and its tests are defined in YAML so they can track benchmark changes

#### Job
A Kubernetes Job is a Kubernetes object used to run a task that should finish successfully, not run forever.
Normal apps in Kubernetes usually run as Deployments:
```text
Deployment = keep my app running all the time
Job        = run this task, finish, then stop
```
Job lifecycle:
A Kubernetes Job does this:
```text
1. Creates one or more Pods
2. Runs the command inside the Pod
3. Waits until the command exits successfully
4. Marks the Job as Completed
5. Keeps the logs so you can inspect them

# Example:
Job: kube-bench
   └── Pod: kube-bench-xxxxx
          └── runs security scan
          └── prints result in logs
          └── exits
```

**Example Kubernetes Job YAML**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: hello
          image: busybox
          command: ["sh", "-c", "echo Hello from Kubernetes Job && sleep 5"]
```

```bash
kubectl apply -f job.yaml
kubectl get jobs
kubectl get pods
kubectl logs job/hello-job
kubectl delete job hello-job
```

**Kubernetes Job = run a one-time task and stop when it finishes.**

## 2. Security Primitives
### Authentication
- who can access Kubernetes - Authentication only proves identity.
- Cluster access is performed through the Kubernetes API Server. Users or clients can communicate with the API Server either by using the `kubectl` tool or by sending direct API requests `curl https://<kube-api-server-ip>:6443/`
	- The API Server is the main entry point to the Kubernetes cluster.
	- Anything you do with `kubectl` goes to the API Server: `kubectl get pods` `kubectl create deployment nginx`
	- Also, internal components communicate with the API Server: kubelet, controller-manager, scheduler, pods using ServiceAccounts
	- Common authentication methods with API Server:
		1. Certificates
		2. ServiceAccounts
		3. External identity providers: LDAP / OIDC / IAM
		4. Static token file - old/simple method, not recommended for production

#### Anatomy
```text
Human users      → external identity
ServiceAccounts  → Kubernetes-native identity
```

##### 1. Normal users
- Users Don't exist as Kubernetes API objects - Kubernetes does not have a resource called User, so you cannot add and view them through the Kubernetes API like `kubectl create user` or `kubectl view users`. 
- A human user usually comes from an external authentication source.

> [!example]
> - [[#^stf|^Static token file]], old method
> - [[#^crt|^Certificates]]
> - LDAP / Active Directory
> - OIDC provider
> - Cloud IAM, like AWS IAM, GCP IAM, Azure AD

- The Kubernetes API Server only receives an identity after authentication.

###### 1. Static Token , Not Recommended ^stf 

Note Recommended But it is important to understand how the Curl command with Bearer token works.
Create token file on master/control-plane node: 
```sh
sudo vi /etc/kubernetes/user-token-details.csv
```
Add users (Format: token,username,user-id,group):
```csv
KpjCVbI7rCFAHYPkBzRb7gu1cUc4B,user10,u0010,group1
rJjncHmvtXHc6MlWQddhtvNyyhgTdxSC,user11,u0011,group1
mjp0FIEiFOKL9toikaRNtt59ePtczZSa,user12,u0012,group2
PG41IXhs7QjqwWkmBkvgGT9glOyUqZij,user13,u0013,group2
```

Use the token from the CSV file:
```sh
curl -v -k https://<master-node-ip>:6443/api/v1/pods \
  --header "Authorization: Bearer KpjCVbI7rCFAHYPkBzRb7gu1cUc4B"
  # --header "Authorization: Bearer <Token>"
```

###### 2. TLS Certificates ^crt

**1. Server creates a private key**
First, the server generates a private key.
```sh
openssl genrsa -out server.key 2048
```
This key must stay secret on the server. `server.key = private key` Never share it.

**2. Server creates a CSR**
CSR means: `Certificate Signing Request`
The server uses the private key to create a request for a certificate.
```sh
openssl req -new \
  -key server.key \
  -out server.csr \
  -subj "/CN=api.example.com"
```

> [!example] The CSR contains information like:
> - Common Name: api.example.com
> - Public Key: server public key
> - Organization info
> - SAN names - Subject Alternative Names, SANs = the list of DNS names or IP addresses that the certificate is valid for

> [!important] modern TLS depends mainly on SAN, not only CN.
> Example SANs:
> - DNS: api.example.com
> - DNS: www.api.example.com
> - IP: 192.168.1.10

**3. Certificate Authority signs the CSR**
A CA signs the CSR and issues a certificate.
> [!example] Examples of public CAs:
> - Let's Encrypt
> - DigiCert
> - GlobalSign
> - Sectigo

> [!example] Examples of internal/private CAs:
> - Company CA
> - Kubernetes Cluster CA
> - Vault PKI
> - cert-manager CA issuer

The CA signs the CSR and returns: `server.crt`
Now the server has:
```text
server.key  → private key
server.crt  → signed certificate
```
The certificate proves:
```text
This public key belongs to api.example.com,
and this claim is trusted because a CA signed it.
```

**4. Server installs the certificate**
Example with Nginx:
```Nginx
server {
    listen 443 ssl;
    server_name api.example.com;

    ssl_certificate     /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;
}
```
The server presents this certificate to clients during the TLS handshake.

**5. Client connects to the server**
- Client opens: `https://api.example.com`
- The server sends its certificate: `server.crt`
- The client checks:
	1. Is the certificate signed by a trusted CA? ==so The client needs a copy of the CA certificate, but only the public CA certificate, not the CA private key.==
	2. Is the certificate still valid?
	3. Does the certificate match api.example.com?

- If all checks pass, the client trusts the server.

**6. Client and server create encrypted session**
After the client trusts the certificate, TLS creates **a shared encryption key.**
The client and server perform a TLS handshake to create a **shared secret key** `Client + Server → create temporary session key`

> [!important] The certificate is not used to encrypt all traffic directly, Instead: 
> - Certificate proves server identity.
> - TLS handshake creates session keys (Shared Key)
> - Both client and server now have the same session key.
> - Same key is used to encrypt and decrypt the traffic.

> [!info] Final flow
> Client connects
>       ↓
> Server sends certificate
>       ↓
> Client verifies certificate
>       ↓
> TLS handshake creates session key
>       ↓
> Encrypted HTTPS communication starts

**What is inside a TLS certificate?**
A certificate usually contains:
```text
Subject:
  Who owns this certificate

Issuer:
  Who signed this certificate

Public Key:
  Server public key

SAN:
  Valid DNS names and IPs

Validity:
  Not before / not after

Key Usage:
  What this certificate can be used for

Digital Signature:
  CA signature
```

> [!example]
```sh
openssl x509 -in server.crt -text -noout
```
> [!example] You may see:
```text
Subject: CN=api.example.com
Issuer: CN=Let's Encrypt Authority
Validity:
  Not Before: ...
  Not After: ...
Subject Alternative Name:
  DNS:api.example.com
Key Usage:
  Digital Signature, Key Encipherment
Extended Key Usage:
  TLS Web Server Authentication
```

**TLS in Kubernetes**
**1. Certificate Authority (CA)**
```sh
# 1. Generate CA Private Key
openssl genrsa -out ca.key 2048
# output file: ca.key => CA private key
# This key is used by the Certificate Authority to sign certificates.

# 2. Create Certificate Signing Request
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
# Output file: ca.csr => Certificate Signing Request

# 3. Self-Sign the CA Certificate
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
# Output file: ca.crt => CA public certificate
# This certificate is used by clients to verify certificates signed by this CA.
```

==For all other certificates, we will use this CA key pair to sign them , the CA has its private key and root certificate file==

**2. Client Certificate**
> [!example] Need to create a certificate  for `admin user`:
```sh
# 1. Create a private key 
openssl genrsa -out admin.key 2048
# admin.key

# 2. Create CSR: username: kube-admin , group: system masters = admin group , with all permissions
openssl req -new -key admin.key -subj \
	"/CN=kube-admin/OU=system:masters" -out admin.csr
# admin.csr

# For example, for the kube-scheduler component, use:
# CN (Common Name) =system:kube-scheduler
# Kubernetes will authenticate this certificate as the user system:kube-scheduler, and RBAC will decide what this user is allowed to do.
openssl req -new \
  -key scheduler.key \
  -subj "/CN=system:kube-scheduler" \
  -out scheduler.csr

# 3. Sign the Certificate using The CA Certificate and The CA Key
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt 
# You're signing your certificate with the CA key pair.
# admin.crt
```

curl command with kube-apiserver using `admin-user` :
```sh
curl https://kube-apiserver:6443/api/v1/pods \
  --key admin.key \
  --cert admin.crt \
  --cacert ca.crt
```

> [!important] The other way is to move all of these parameters into configs file called [[#^kubeconfig]] with specify the API Server Endpoint

> [!example] output
```json
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/pods"
  },
  "items": []
}
```

> [!important] Flow
> curl connects to kube-apiserver
>         ↓
> curl verifies API Server certificate using ca.crt
>         ↓
> curl sends admin.crt as client certificate
>         ↓
> API Server verifies admin.crt
>         ↓
> API Server authenticates the request as the certificate user
>         ↓
> RBAC checks permissions
>         ↓
> API returns PodList 

> [!note]
> - In Kubernetes, core components use certificates and private keys to securely communicate with each other. Components that connect to the API Server use client certificates, while components that receive connections use server certificates.
> - etcd can run as one node or multiple nodes so etcd has an additional Certificated called **peer certificate** used for secure communication between etcd members in a multi-node etcd cluster.

> [!note] Problem and Solution: Kube API Server Certificate
> The API Server can be accessed using different names:
> - https://kubernetes
> - https://kubernetes.default
> - https://kubernetes.default.svc
> - https://kubernetes.default.svc.cluster.local
> - https://10.96.0.1
> - https://172.17.0.87
> 
> So all these names/IPs must exist inside the certificate SANs so will use [[#^openssl|^openssl.cnf]] file as shown below

```sh
# Generate CSR for kube-apiserver
openssl req -new -key apiserver.key \
  -subj "/CN=kube-apiserver" \
  -out apiserver.csr \
  -config openssl.cnf
```

**openssl.cnf** ^openssl

```text
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name

[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation
subjectAltName = @alt_names

[alt_names] ###################################### All Names
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.0.1
IP.2 = 172.17.0.87
```

```sh
openssl x509 -req \
  -in apiserver.csr \
  # Remember, every component needs the CA certificate to verify its clients: 
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out apiserver.crt \
  -extensions v3_req \
  # mentioned the openssl config file: 
  -extfile openssl.cnf \
  -days 1000
```

**kube-apiserver config lines**
```sh
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --enable-swagger-ui=true \\

  # CA certificate used by kube-apiserver to verify the etcd server certificate
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\

  # API Server needs to talk with ETCD (API Server is a Client for ETCD) , so this certificate used by kube-apiserver to authenticate itself to etcd
  --etcd-certfile=/var/lib/kubernetes/apiserver-etcd-client.crt \\

  # Private key for the kube-apiserver etcd client certificate
  --etcd-keyfile=/var/lib/kubernetes/apiserver-etcd-client.key \\

  --etcd-servers=https://127.0.0.1:2379 \\
  --event-ttl=1h \\

  # CA certificate used by kube-apiserver to verify the kubelet server certificate
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\

  # API Server needs to talk with Kublete (API Server is a Client for Kubelet) Client certificate used by kube-apiserver to authenticate itself to kubelet
  --kubelet-client-certificate=/var/lib/kubernetes/apiserver-kubelet-client.crt \\

  # Private key for the kube-apiserver kubelet client certificate
  --kubelet-client-key=/var/lib/kubernetes/apiserver-kubelet-client.key \\

  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\

  # CA certificate used by kube-apiserver to verify incoming client certificates
  # Examples: admin, kubelet, scheduler, controller-manager, kube-proxy
  --client-ca-file=/var/lib/kubernetes/ca.pem \\

  # Server certificate used by kube-apiserver to prove its identity to clients
  --tls-cert-file=/var/lib/kubernetes/apiserver.crt \\

  # Private key for the kube-apiserver server certificate
  # is used when the API Server acts as a server.
  --tls-private-key-file=/var/lib/kubernetes/apiserver.key \\

  --v=2
```

> [!note]
> - When kube-apiserver connects to kubelet, the API Server acts as a client and the kubelet acts as a server.
> - First, the API Server verifies the kubelet server certificate using the **CA** configured with `--kubelet-certificate-authority`
> - Then, the API Server authenticates itself to the kubelet using the Certificate`--kubelet-client-certificate` and the Private key`--kubelet-client-key`
> - The kubelet verifies whether the API server client certificate <u>is valid or not</u> by using its `clientCAFile`
> - Then kubelet checks that the API Server <u>owns the certificate</u> by verifying a signature created with the API Server client private key `--kubelet-client-key`. The kubelet verifies this signature using the public key inside the API Server client certificate `--kubelet-client-certificate` - The API Server does not send the private key, API Server takes TLS handshake data **(during the TLS handshake, both sides exchange messages before encryption starts. These messages are the handshake data)** , and signs it using its private key **(API Server uses its private key to create a digital signature over the handshake data)**, Then API Server sends the signature to kubelet.
> - kubelet already has the certificate `--kubelet-client-certificate` and inside that certificate there is the public key, This signature proves The API Server owns the private key that matches the public key inside its certificate.
> - After this TLS handshake, both sides derive **symmetric session keys**, and these session keys encrypt the actual traffic.

**Kubelete Server - Kubectl Nodes (Client Certificates)**
- The kubelet server is an HTTPS API server that runs on each node, responsible for managing the node. That's who the API server talks to, to monitor the node as well as send information regarding what pods to schedule on this node So the kubelet needs a server certificate to prove its identity to the API Server.
- As such, you need a key/certificate pair for each node in cluster - A node certificate means the kubelet server certificate for each Kubernetes node.

After Create private key and certificate for each node, In the kubelet config:
```yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  x509:
	# CA Certificate
    clientCAFile: "/var/lib/kubernetes/ca.pem" 
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"

# for each node: 

tlsCertFile: "/var/lib/kubelet/kubelet-node01.crt"
# kube-apiserver connects to kubelet and verifies this certificate

tlsPrivateKeyFile: "/var/lib/kubelet/kubelet-node01.key"
# Private key for the kubelet server certificate
```

> [!important]
> - CN=`system:node:node01` and O=`system:nodes` are very important because the API Server uses them for authentication and Node Authorization
> - config: -subj "/CN=system:node:node01/O=system:nodes"

> [!tip] 
> View Certificate Details: 

```sh
cat /etc/kubernetes/manifests/kube-apiserver.yaml
# .....
# - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
# .....

openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```

> [!example] Output

```text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 3147495682089747350 (0x2bae26a58f090396)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=kubernetes ##########################
        Validity
            Not Before: Feb 11 05:39:19 2019 GMT
            Not After : Feb 11 05:39:20 2020 GMT ##########################
        Subject: CN=kube-apiserver ##########################
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:d9:69:38:80:68:3b:b7:2e:9e:25:00:e8:fd:01:
                Exponent: 65537 (0x10001)

        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
            X509v3 Subject Alternative Name: ##########################
                DNS:master,
                DNS:kubernetes,
                DNS:kubernetes.default,
                DNS:kubernetes.default.svc,
                DNS:kubernetes.default.svc.cluster.local,
                IP Address:10.96.0.1,
                IP Address:172.17.0.27
```

```sh
# Service Logs:

# if you setup cluster from scratch and services are configured as native service in the OS
journalctl -u etcd.service

# Cluster with kubeadm: various components are deployed as pods.
kubectl logs etcd-master

# If the core components such as the kube-apiserver or the ETCD are down , the kubectl command won't function, in that case , you have to go one level down to docker to fetch the logs as shown below
# Container-d is commonly the default/recommended runtime, Use the Commands blow to check containerd logs: 
crictl ps -a
crictl logs #<etcd-master Container id>
```

**Certificates API**
- The CA is the pair of key and certificate files we have generated. Whoever gains access to this pair of files can sign any certificate for the Kubernetes environment; they can create as many users as they want, with whatever privileges they want. So these files need to be protected and stored in a safe environment.
- So we place them on a server that is fully secure (CA server).
- The certificate key file is safely stored in that server, Every time you want to sign a certificate, you can only do it by logging into that server.
- we have the certificates placed on the Kubernetes master node itself. So the master node is our CA server. The `kubeadm` tool does the same thing. It creates a CA pair of files and stores that on the master node itself. 
- So far, we have been signing requests manually. But you need a better automated way to manage the certificates, signing requests

Certificate API means the Kubernetes API group (certificates.k8s.io) :
```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
```
`CertificateSigningRequest` is Kubernetes’ built-in way to ask the cluster to sign a certificate, then the request can be approved or denied before it is signed

1. Generate a Private key and a CSR 
```sh
$ openssl genrsa -out jane.key 2048
jane.key

$ openssl req -new -key jane.key -subj "/CN=jane" -out jane.csr
jane.csr

-----BEGIN CERTIFICATE REQUEST-----
MIICWDCCAUACAQAwEzERMA8GA1UEAwwIbmV3LXVzZXIwggEiMA0GCSqGSIb3DQEB
AQUAA4IBDwAwggEKAoIBAQD0OWJw+DXsAJSIrjpNo5vRIBplnzg+6xc9+UVwkKi0
LfC27t+1eEnON5Muq99NevmMEOnrDUO/thyVqP2w2XNIDRXjYyF40FbmD+5zWyCK
9w0BAQsFAAOCAQEAS9iS6C1uxTuf5BBYSU7QFQHUza1NxAdYsaORRQNwHZwHqGi4
hOK4a2zyNyi44OOijyaD6tUW8DSxkr8BLK8Kg3srREtJq15rLZy9LRVrsJghD4gY
P9NL+aDRSxROVSqBaB2nWeYpM5cJ5TF53lesNSNMLQ2++RMnjDQJ7juPEic8/dhk
Wr2EUM6UawzykrdHImwTv2m1MY0R+DNtV1Yie+0H9/YE1t+FSGjh5L5YUvI1Dqiy
4l3E/y3qL71WfAcuH3OsVpUUnQISMdQs0qWCsbE56CC5DhPGZIpUbnKUpAwka+8E
vwQ07jG+hpknxmuFAeXxgUwodALaJ7ju/TDIcw==
-----END CERTIFICATE REQUEST-----
```
Then sends the request to the administrator.
2. The administrator takes the `certificate signing request` and creates a `CertificateSigningRequest` object.
```sh
cat jane.csr | base64 -w 0
```

jane-cr.yaml: 
```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: myuser # example
spec:
  # This is an encoded CSR. Change this to the base64-encoded contents of myuser.csr
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBNQTBHQTFVRUF3d0dZVzVuWld4aE1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJDZ0tDQVFFQTByczhJTHRHdTYxakx2dHhWTTJSVlRWMDNHWlJTWWw0dWluVWo4RElaWjBOCnR2MUZtRVFSd3VoaUZsOFEzcWl0Qm0wMUFSMkNJVXBGd2ZzSjZ4MXF3ckJzVkhZbGlBNVhwRVpZM3ExcGswSDQKM3Z3aGJlK1o2MVNrVHF5SVBYUUwrTWM5T1Nsbm0xb0R2N0NtSkZNMUlMRVI3QTVGZnZKOEdFRjJ6dHBoaUlFMwpub1dtdHNZb3JuT2wzc2lHQ2ZGZzR4Zmd4eW8ybmlneFNVekl1bXNnVm9PM2ttT0x1RVF6cXpkakJ3TFJXbWlECklmMXBMWnoyalVnald4UkhCM1gyWnVVV1d1T09PZnpXM01LaE8ybHEvZi9DdS8wYk83c0x0MCt3U2ZMSU91TFcKcW90blZtRmxMMytqTy82WDNDKzBERHk5aUtwbXJjVDBnWGZLemE1dHJRSURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBR05WdmVIOGR4ZzNvK21VeVRkbmFjVmQ1N24zSkExdnZEU1JWREkyQTZ1eXN3ZFp1L1BVCkkwZXpZWFV0RVNnSk1IRmQycVVNMjNuNVJsSXJ3R0xuUXFISUh5VStWWHhsdnZsRnpNOVpEWllSTmU3QlJvYXgKQVlEdUI5STZXT3FYbkFvczFqRmxNUG5NbFpqdU5kSGxpT1BjTU1oNndLaTZzZFhpVStHYTJ2RUVLY01jSVUyRgpvU2djUWdMYTk0aEpacGk3ZnNMdm1OQUxoT045UHdNMGM1dVJVejV4T0dGMUtCbWRSeEgvbUNOS2JKYjFRQm1HCkkwYitEUEdaTktXTU0xMzhIQXdoV0tkNjVoVHdYOWl4V3ZHMkh4TG1WQzg0L1BHT0tWQW9FNkpsYWFHdTlQVmkKdjlOSjVaZlZrcXdCd0hKbzZXdk9xVlA3SVFjZmg3d0drWm89Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
```

```sh
kubectl apply -f jane-cr.yaml
kubectl get csr
kubectl certificate approve #<certificate name>
```
Kubernetes signs the certificate using the **CA key pair (CA server's root certificate and private key)** and generates a certificate for the user.
This certificate can then be extracted and shared with the user.

```sh
kubectl get csr jane -o yaml
# ....
# status:
#   certificate:
#     LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURDakNDQWLZOIF3SUJBZ0lVRmwyQ2wx
#     YXoxawl5M3JNV1sreFRYQUowU3dnd0RRWUpLb1JHaHJjTkFRRUwKQlFBA0ZURVRMQkVHQTFV
#     RUF4UTHM1ZpWlhKdVpYUmxjKEF1RncwEE9UQX1NVE14TmpNeU1EQmFGd1dNY0ZFeD12ajNu
#     SXY3eFdDS1NIRm5sU041c0t5Z0VxUkwzTFM5V29GelhHZDdWCM1EZ2FOMVVRMFBXTVhjN09F
#     VnVjSwc1Yk4weEVHTkVwRUStdU1BN1ZWeHVjS1h6aG91dDY0MEd1MGU0YXFkWVIKWmVMbjBv
#     RTFCY3dod2xic0I1ND0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
# ....

# it's in a base64 encoded format to decoded: 
echo "LS0.." | base64 --decode
# now can share it with enduser
```

> [!important] 
> - The controller manager is actually responsible for all the certificate-related operations. If you look at the controller manager, you will see that it has controllers in it called `CSR approving`, `CSR signing`, etc. They’re responsible for carrying out these specific tasks
> - We know that if anyone has to sign certificates, they need the `CA server's root certificate` and `private key` as shown below

```sh
$ cat /etc/kubernetes/manifests/kube-controller-manager.yaml
    spec:
      containers:
      - command:
        - kube-controller-manager
        - --address=127.0.0.1
		
		# CA Certificate and Private key	
        - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
        - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
		
        - --controllers=*,bootstrapsigner,tokencleaner
        - --kubeconfig=/etc/kubernetes/controller-manager.conf
        - --leader-elect=true
        - --root-ca-file=/etc/kubernetes/pki/ca.crt
        - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
        - --use-service-account-credentials=true
```

**KubeConfig File** ^kubeconfig

A kubeconfig file is the configuration file that tells kubectl:
- Which Kubernetes cluster to connect to
- Which user credentials to use
- Which namespace to use by default

cat `~/.kube/config` :
```yaml
apiVersion: v1
kind: Config

clusters:
- name: my-cluster
  cluster:
	# server ip and its CA certificate
    server: https://1.2.3.4:6443
    certificate-authority: /etc/kubernetes/pki/ca.crt

users:
- name: jane
  user:
	# User Certificate and Private Key
    client-certificate: /etc/kubernetes/pki/users/jane.crt
    client-key: /etc/kubernetes/pki/users/jane.key

contexts:
- name: jane-context
  context:
    cluster: my-cluster
    user: jane
    namespace: dev

# Current Context
current-context: jane-context
```

- clusters       = API server connection info
- users          = authentication info
- contexts       = cluster + user + namespace combination
- current-context = the context kubectl is currently using

```sh
kubectl config view
# by default kubectl will use kubeconfig file in ~/.kube/config
kubectl config view --kubeconfig #<config file path>

# switch from one Kubernetes context to another:
kubectl config get-contexts
kubectl config use-context <context-name>
```

> [!example]

```sh
kubectl config get-contexts
```

```text
CURRENT   NAME                 CLUSTER        AUTHINFO      NAMESPACE
*         admin-context        my-cluster     admin
          jane-context         my-cluster     jane          dev
```

```sh
# Switch to jane-context:
kubectl config use-context jane-context

# Check current context:
kubectl config current-context
```

##### 2. Service Accounts
- You can create and manage service accounts using the Kubernetes API, so we can do below commands : 
```sh
kubectl create serviceaccount sa1 -n production
kubectl get serviceaccount -n production
# Short version:
kubectl get sa -n production
```

- In order for my application to query the Kubernetes API, it has to be authenticated , for that we use a service account.
- A service account is the identity of a service.
- A service account is associated with what is known as a token.
- so if you authenticated to an API using a curl command or something , then the token is passed through as a Bearer token in the header of the call, this token is what proves the identity of the SA to the API endpoint 

###### Service Account and Token
By default, when a Kubernetes cluster is created, Kubernetes creates a default ServiceAccount in every namespace.
To list existing ServiceAccounts, use:
```sh
kubectl get serviceaccount
```
Example output:
```text
NAME      SECRETS   AGE
default   1         218d
```
To see the details of an existing ServiceAccount, run:
```sh
kubectl describe serviceaccount default
```
Example output:
```text
Name:                default
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   <none>
Tokens:              <none>
Events:              <none>
```
Whenever a Pod is created, if you do not specify a custom ServiceAccount, Kubernetes automatically attaches the **default ServiceAccount** to the Pod.
The default ServiceAccount has limited permissions. So if your app needs specific access, create a [[#^CustomServiceAccount|^Custom Service Account]]

You can check this by describing the Pod:
```sh
kubectl describe pod my-kubernetes-dashboard
```
> [!example] output
```text
Name:             my-kubernetes-dashboard
Namespace:        default
Priority:         0
Service Account:  defaulat ##########################
Node:             controlplane/192.168.96.172
Start Time:       Wed, 03 Sep 2025 16:30:28 +0000
Labels:           name=web-dashboard
                  pod-template-hash=5f88cdc488

Containers:
  web-dashboard:
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount

Volumes:
  kube-api-access-rckvd:
    Type:                    Projected
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
```
> [!attention] 
 > Kubernetes mounts the ServiceAccount token inside the Pod as a projected volume at: `/var/run/secrets/kubernetes.io/serviceaccount`
> - Projected Volume = a Kubernetes volume that combines multiple sources (Multi Files: token && ca.crt && namespace) into one mount.

```sh
kubectl exec -it my-kubernetes-dashboard -- ls /var/run/secrets/kubernetes.io/serviceaccount
```

```text
ca.crt  namespace  token
```

> [!attention]
> - Inside the Pod, the application can use this **token** to communicate with the Kubernetes API Server, if it has the required RBAC permissions.

**Create a Custom Service Account** ^CustomServiceAccount

Create a SA using command: 
```sh
kubectl create serviceaccount dashboard-sa
```
Create a SA using Yaml file: 
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-sa
  namespace: default
```

```sh
kubectl get serviceaccount
kubectl describe serviceaccount dashboard-sa
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  containers:
    - name: my-kubernetes-dashboard
      image: my-kubernetes-dashboard
  serviceAccountName: dashboard-sa
```
When Describe the Pod: 
```sh
kubectl describe pod my-kubernetes-dashboard
```
> [!example] output
```text
Name:             my-kubernetes-dashboard
Namespace:        default
Priority:         0
Service Account:  dashboard-sa ########################
Node:             controlplane/192.168.96.172
Start Time:       Wed, 03 Sep 2025 16:30:28 +0000
Labels:           name=web-dashboard
                  pod-template-hash=5f88cdc488

Containers:
  web-dashboard:
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount

Volumes:
  kube-api-access-rtvkd:
    Type:                    Projected
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
```
- This confirms that the Pod is now using the custom ServiceAccount dashboard-sa ( as a Projected Volume ) instead of the default ServiceAccount.
- Kubernetes [[#^automatically]] creates a **short-lived token** and **mounts** it inside a projected volume inside a Pod.
- The **kubelet** also automatically rotates this token, and this token is bound to the life of the pod, so it is automatically expired when the pod is deleted.

**Disable Auto-Mounting of ServiceAccount Token** ^automatically

 If you do not want this token to be automatically mounted inside the Pod, you can set: `automountServiceAccountToken: false`
 You can define this in two places:
1. At the ServiceAccount level
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  containers:
    - name: my-kubernetes-dashboard
      image: my-kubernetes-dashboard
  serviceAccountName: dashboard-sa
  automountServiceAccountToken: false ######
```
This specific Pod will not automatically get the ServiceAccount token mounted.

2. At the Pod level
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-sa
  namespace: default
automountServiceAccountToken: false  #####
```
or using patch command: 
```sh
kubectl patch sa dashboard-sa -p '{"automountServiceAccountToken": false}'
```
Any Pod using dashboard-sa will not automatically get the ServiceAccount token mounted. 

**Create a Token**
Creating a token for an external application running outside the cluster: 
```sh
# the ServiceAccount dashboard-sa must already exist, Then Create a Token with the same name: 
kubectl create token dashboard-sa
# The expiration time is 1H

# create a token with a custom expiration time
kubectl create token dashboard-sa --duration 2h
```
You can use the generated token to call the Kubernetes API Server:
```sh
TOKEN="<paste-token-here>"

curl -k https://<kube-api-server-ip>:6443/api/v1/pods \
  --header "Authorization: Bearer $TOKEN"
```

> [!summary]
> 1. Kubernetes creates a default ServiceAccount in every namespace.
> 2. If a Pod does not specify a ServiceAccount, it uses the default one.
> 3. The ServiceAccount token is mounted inside the Pod.
> 4. The token allows the Pod to authenticate to the Kubernetes API Server.
> 5. RBAC decides what this ServiceAccount is allowed to do.

---

### Authorization
#### Prerequisites - API Groups
- API group is a way to organize Kubernetes resources inside the Kubernetes API based on their purpose.
- Such as one for APls `/apis`, one for health `/healthz`, one for metrics`/metrics` and logs`logs`, etc.
```sh
$ curl https://kube-master:6443/version
{
  "major": "1",
  "minor": "33",
  "gitVersion": "v1.33.0",
  "gitCommit": "ddf47ac13c1a9483ea035a79cd7c10005ff21a6d",
  "gitTreeState": "clean",
  "buildDate": "2018-12-03T20:56:12Z",
  "goVersion": "go1.11.2",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

- `/metrics`  && `/healthz` are used to monitor the health of the cluster.
- `/logs` The logs APl is used for integrating with third-party logging applications.

APls Responsible For The Cluster Functionality, these APIs are categorized into two:
**1. Core Group** `/api`
- The core group is where all core functionality exists.
- such as `namespaces`, `pods`, `replication controllers`, `events`, `endpoints`, `nodes`, `bindings`, `persistent volumes`, `persistent volume claims`, `ConfigMaps`, `secrets`, `services`, etc.

**2. Named Group** `/apis`
- The named group APls are more organized.
- All the newer features are going to be made available through these named groups.
- such as `apps`, `batch`, `networking.k8s.io`, `rbac.authorization.k8s.io` , `certificates.k8s.io`, `storage.k8s.io`, `autoscaling`, `apiextensions.k8s.io`
- Named Group => API Groups => Resources => Verbs - Actions
![api named group](./assets/named_group.png)

> [!example]

```sh
curl https://localhost:6443/ -k 
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```
- The `-k` option means: Ignore TLS certificate verification, So `curl` will connect even if the API server certificate is self-signed or not trusted by your machine.
- To Fix Unauthorized error, we need to specify the Private key , Certificate and CA Using command flags: curl https://localhost:6443 -k `--key admin.key` `--cert admin.crt` `--cacert ca.crt`
- An alternate option is to start a kubectl proxy client using commad: `kubectl proxy`,<u>this launchesa a **local HTTP proxy** from your machine to the **Kubernetes API Server** on port `8001`</u>, and uses credentials from your kubeconfig file to access the cluster. That way, you don't have to specify those in the curl command.
- **Kube-Proxy vs Kubectl Proxy**: 
	- kube-Proxy: is used to enable connectivity between pods and services across different nodes in the cluster.
	- Kubectl Proxy: is an HTTP proxy service created by kubectl utility to access the kube-apiserver.

```sh
# 0. Launch local http proxy
kubectl proxy  # --port=<Port Number>
# output: Starting to serve on 127.0.0.1:8001

# 1. This command shows the main API paths available on the Kubernetes API server.
# Use port: 8001 instead of 6443
curl http://localhost:8001/ -k 
{
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/healthz",
    "/logs",
    "/metrics",
    "/openapi/v2",
    "/swagger-2.0.0.json",
    ...
    ...
    ...
  ]
}

# 2. This command lists the named API groups available in the cluster.
curl http://localhost:8001/apis -k | grep "name"

"name": "extensions",
"name": "apps",
"name": "events.k8s.io",
"name": "authentication.k8s.io",
"name": "authorization.k8s.io",
"name": "autoscaling",
"name": "batch",
"name": "certificates.k8s.io",
"name": "networking.k8s.io",
"name": "policy",
"name": "rbac.authorization.k8s.io",
"name": "storage.k8s.io",
"name": "admissionregistration.k8s.io",
"name": "apiextensions.k8s.io",
"name": "scheduling.k8s.io",
"name": "coordination.k8s.io",

```

**Chatgpt Notes**
> [!example]

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest

# certificates.k8s.io = API group
# v1                  = API version
# CertificateSigningRequest = resource kind
```
So this resource belongs to the certificates.k8s.io API group.
```text
apiVersion: <api-group>/<version>
```

> [!example]

```yaml
apiVersion: apps/v1
kind: Deployment

# API group: apps
# Version: v1
# Kind: Deployment
```

There is also the `core API group`, which has no group name. For core resources, the `apiVersion` is just:
```yaml
apiVersion: v1
kind: Pod
```
Other core resources include:
- Pod
- Service
- ConfigMap
- Secret
- Namespace
- Node
- PersistentVolume
- PersistentVolumeClaim

`apiVersion: v1` means it is in the core API group.

A simple way to think about it:
- API group = category of Kubernetes resources
- API version = version of that category
- Kind = the object type

---

#### Anatomy
What are you allowed to do?
- Authorization modes in Kubernetes: [[#^Node]] , [[#^ABAC]] , [[#^RBAC]] , [[#^Webhook]]
- if we have: `--authorization-mode=Node,RBAC,Webhook`, the decision rules are:
	- ALLOW => stop checking, request is allowed
	- DENY => stop checking, request is denied
	- NO OPINION =>  move to the next authorization mode

##### Types

**1. Node Mode** ^node
- Node Authorization is mainly for kubelet.
- The kubelet runs on every node and talks to the API Server.
- Example kubelet identity:
	- Username = `system:node:node01`
	- Group    = `system:nodes`
- Node Authorization allows the kubelet to do only node-related actions, for example:
	- Read its own Node object
	- Update its own Node status
	- Read Pods scheduled on its node
	- Read Secrets/ConfigMaps needed by Pods on its node
	- Report Pod status
- It should not allow kubelet on node01 to freely access everything in the cluster. 
- Example: 
	- **node01** kubelet can access pods scheduled on **node01**
	- **node01** kubelet should not control pods on **node02**
- Usually enabled like this:
	- `--authorization-mode=Node,RBAC`
	- `--enable-admission-plugins=NodeRestriction`
		- `--authorization` = special permissions for kubelet nodes.
		- Without `Node Restriction`, a kubelet might be able to say: "I am node-1, but I want to modify node-2 or update pods not running on me." `NodeRestriction` blocks that.

**2. ABAC** ^abac
- Attribute-Based Access Control
- It uses a [[#^spf|^static policy file]]

**Example policy** ^spf

```text
{
  "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
  "kind": "Policy",
  "spec": {
    "user": "omar",
    "namespace": "default",
    "resource": "pods",
    "readonly": true
  }
}
```
This means: **User omar can read pods in the default namespace.**
But ABAC is not commonly used now because:
- Policy is file-based
- Changes require API Server restart/reload
- Harder to manage
- Not as flexible as RBAC

**3. Webhook** ^webhook
- Webhook Authorization means the API Server asks an external authorization service whether to allow or deny the request.
- This is useful when a company wants authorization decisions to be handled by an external system.

**4. RBAC** ^rbac
- Role-Based Access Control
- In Kubernetes, the most common authorization mode is **RBAC** Role-Based Access Control
- Two types: [[#^ns|^Namespace Scoped]] and [[#^cs|^Cluster Scoped]]

**1. Namespace Scoped** ^ns
- Namespace-scoped RBAC controls who can do what within a single namespace, using two objects: `Role` and `RoleBinding`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
	# apiGroup: [""] , Pod is Core api Group and The core API group has no group name, so apiGroup represents as an empty string ""
  - apiGroups: [""] 
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

> [!important] But this **only protects** the **API Server**
> - Authentication and Authorization protect access to the Kubernetes API Server.
> - They control actions like:
> 	- create pod
> 	- delete deployment
> 	- read secret
> 	- update service
> 	- list nodes
> 	- create ingress

> [!important] But they do not automatically control traffic between applications.
> - This is not controlled by RBAC. 
> - **RBAC controls Kubernetes API permissions.**
> - It does not control normal network traffic between pods.
> - By default, all pods can access all other pods within the cluster. you can restrict access between them using [[#Network Policies|^Network Policies]]

> [!example]

Create a service account:
```sh
kubectl create serviceaccount backend-sa -n production
```

Use it in a Deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      serviceAccountName: backend-sa
      containers:
        - name: backend
          image: nginx
```
Now the Pod runs as: **system:serviceaccount:production:backend-sa**

allow backend-sa to read pods only: 
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```
Bind it to the ServiceAccount:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-sa-read-pods
  namespace: production
subjects:
  - kind: ServiceAccount
    name: backend-sa
    namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```sh
# backend-sa can read pods
# backend-sa cannot delete pods
# backend-sa cannot read secrets
# backend-sa cannot create deployments

# Test ServiceAccount permission: --as <User|Service Account>
kubectl auth can-i list pods \
  --as=system:serviceaccount:production:backend-sa \
  -n production
```

> [!important]
> Can limits the permission to only specific pods:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create"]

  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "delete"]
	
	# Resource Names 
    resourceNames: ["blue", "orange"]
```

**2. Cluster-Scoped** ^cs
- Permissions that apply at the whole cluster level, not only inside one namespace
- Controls permissions that span the whole cluster, using two objects: `ClusterRole` and `ClusterRoleBinding`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-administrator
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["list", "get", "create", "delete"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
  - kind: User
    name: cluster-admin
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io
```

---

### Network Policies
**Prerequisites - CNI**
- CNI means Container Network Interface , is the networking system/plugin that gives Pods network connectivity.
- It gives Pods IP addresses and connects them to the cluster network.

> [!info] Simple flow
> Pod is created
>       ↓
> kubelet asks container runtime to create Pod sandbox
>       ↓
> container runtime calls CNI plugin
>       ↓
> CNI plugin creates network interface - creates network connectivity for the Pod
>       ↓
> Pod gets IP address
>       ↓
> Pod can communicate in the cluster

- Solutions that Support Network Policies: `Calico`, `Cilium`, `Kube-router`, `Romana`
- Solutions that DO NOT Support Network Policies: `Flannel`

**Anatomy**
- controlling application communication
- **To restrict traffic between Pods, Which pod is allowed to talk to which pod?**
	- NetworkPolicy = firewall rules for Pods
	- Only backend can access postgres.
	- Frontend cannot access postgres directly.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              name: api-pod
      ports:
        - protocol: TCP
          port: 3306
```
- This NetworkPolicy applies to Pods with label `role=db`
- It allows incoming traffic only from Pods with label `name=api-pod`
- It allows traffic only on TCP port `3306`, usually MySQL.
- Other incoming traffic to the db Pods will be blocked

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              name: api-pod
      ports:
        - protocol: TCP
          port: 3306
  egress:
    - to:
        - ipBlock:
            cidr: 192.168.5.10/32
      ports:
        - protocol: TCP
          port: 80
```
- DB Pods can receive traffic only from api-pod on port 3306.
- DB Pods can send traffic only to 192.168.5.10 on port 80.
- All other ingress and egress traffic for DB Pods is denied.

---

## 3. Securing The Kubelet
- The kubelet in the Kubernetes worker node registers the node with the Kubernetes cluster.
- When it receives instructions to load a container or a pod on the node, it requests the container runtime engine, which may be Docker or any other runtime engine, to pull the required image and run an instance.
- And the kubelet then continues to monitor the state of the pod and the containers in it and reports the status to the kube-apiserver on a timely basis.
- With the release of version 1.10, most of the parameters Configurations were moved to another file called the [[#^kc|^kubelet-config.yaml]] for ease of deployment and configuration management.

`kubelet.service` :
```sh
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kubelet
ExecStart=/usr/local/bin/kubelet \\
  --container-runtime=docker \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2 \\
  # With the release of version 1.10, most of the parameters Configurations were moved to another file called the kubelet-config.yaml
  # --cluster-domain=cluster.local \\
  # --file-check-frequency=0s \\
  # --healthz-port=10248 \\
  # --cluster-dns=10.96.0.10 \\
  # --http-check-frequency=0s \\
  # --sync-frequency=0s
  
  # pass the path of kubelet-config.yaml  
  --config=/var/1ib/kubelet/kubelet-config.yaml
```

**kubelet-config.yaml** ^kc
```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
clusterDomain: cluster.local
fileCheckFrequency: 0s
healthzPort: 10248
clusterDNS:
  - 10.96.0.10
httpCheckFrequency: 0s
syncFrequency: 0s
```

> [!important] Viewing kubelet Options
```sh
ps -aux | grep kubelet
# ...
  --config=/var/lib/kubelet/config.yaml \      ### config file for kubelet ###
# ...
  
cat /var/lib/kubelet/config.yaml
# apiVersion: kubelet.config.k8s.io/v1beta1
# clusterDNS:
#   - 10.96.0.10
# clusterDomain: cluster.local
# cpuManagerReconcilePeriod: 0s
# evictionPressureTransitionPeriod: 0s
# fileCheckFrequency: 0s
# healthzBindAddress: 127.0.0.1
# healthzPort: 10248
# httpCheckFrequency: 0s
# imageMinimumGCAge: 0s
# kind: KubeletConfiguration
# nodeStatusReportFrequency: 0s
# nodeStatusUpdateFrequency: 0s
# rotateCertificates: true
# runtimeRequestTimeout: 0s
# staticPodPath: /etc/kubernetes/manifests
# streamingConnectionIdleTimeout: 0s
# syncFrequency: 0s
# volumeStatsAggPeriod: 0s
```

How do we make sure that the kubelet only responds to requests from the kube-apiserver and not anyone else?
- The kubelet serves on two ports: 
	- Port `10250`, which is where the kubelet serves its API that allows full access
	- Port `10255`, which serves an API that allows unauthenticated, unauthorized read-only access.
- By default, the kubelet allows anonymous access - *by default Permit all requests without any kind of authentication* - to its API. So if you do a curl to port `10250` at the API `/pods` you'll be able to see the list of pods running on the node.
- Similarly, the service running at port `10255` provides read-only access to metrics to any unauthenticated or unauthorized clients.

> [!example]

```sh
curl -sk https://localhost:10250/pods/
# -s: means silent mode.
# -k: means skip TLS certificate verification.
```

```json
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {},
  "items": [
    {
      "metadata": {
        "name": "kube-proxy-nngzb",
        "generateName": "kube-proxy-",
        "namespace": "kube-system",
        "selfLink": "/api/v1/namespaces/kube-system/pods/kube-proxy-nngzb",
        "uid": "3c5e5745-f67e-4f91-beb9-f11d4da7a0d0",
        "resourceVersion": "631",
        "creationTimestamp": "..."
      }
    }
  ]
}
```

```sh
curl -sk https://localhost:10250/logs/syslog
```

```text
Nov 10 11:26:59 host01 kernel: [    0.000000] Linux version 4.15.0-29-generic
Nov 10 11:26:59 host01 kernel: [    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz-4.15.0-29-generic
Nov 10 11:26:59 host01 kernel: [    0.000000] KERNEL supported cpus:
Nov 10 11:26:59 host01 kernel: [    0.000000] Intel GenuineIntel
Nov 10 11:26:59 host01 kernel: [    0.000000] AMD AuthenticAMD
```

```sh
curl -sk http://localhost:10255/metrics
```

```text
# HELP apiserver_audit_event_total [ALPHA] Counter of audit events generated and sent to the audit backend.
# TYPE apiserver_audit_event_total counter
apiserver_audit_event_total 0

# HELP apiserver_audit_requests_rejected_total [ALPHA] Counter of apiserver requests rejected due to an error in audit logging backend.
# TYPE apiserver_audit_requests_rejected_total counter
apiserver_audit_requests_rejected_total 0

# HELP apiserver_client_certificate_expiration_seconds [ALPHA] Distribution of the remaining lifetime on the certificate used to authenticate a request.
# TYPE apiserver_client_certificate_expiration_seconds histogram
apiserver_client_certificate_expiration_seconds_bucket{le="0"} 0
apiserver_client_certificate_expiration_seconds_bucket{le="1800"} 0
apiserver_client_certificate_expiration_seconds_bucket{le="3600"} 0
apiserver_client_certificate_expiration_seconds_bucket{le="7200"} 0
apiserver_client_certificate_expiration_seconds_bucket{le="21600"} 0
apiserver_client_certificate_expiration_seconds_bucket{le="43200"} 0
apiserver_client_certificate_expiration_seconds_bucket{le="86400"} 0
apiserver_client_certificate_expiration_seconds_bucket{le="172800"} 0
apiserver_client_certificate_expiration_seconds_bucket{le="345600"} 0
apiserver_client_certificate_expiration_seconds_bucket{le="604800"} 0
apiserver_client_certificate_expiration_seconds_bucket{le="2.592e+06"} 0
apiserver_client_certificate_expiration_seconds_bucket{le="7.776e+06"} 0
apiserver_client_certificate_expiration_seconds_bucket{le="1.5552e+07"} 0
```

**This is a big security risk**, How do you implement security on the kubelet?
- Any request that comes to the kubelet is first authenticated and then authorized.
	- The [[#^auth|^authentication]] process decides if the user or the requesting entity has access of API
	- The [[#^autho|^authorization]] process decides what areas of the API the user can access and what operations they can perform.

### Authentication ^auth
Need to disables anonymous access to the kubelet API

`kubelet.service` :
```sh
ExecStart=/usr/local/bin/kubelet \\
#  ...
  --anonymous-auth=false
#  ...
```

or if we use `kubelet-config.yaml`: 
```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  anonymous:
    enabled: false
```

- We create a pair of certificates that will be used as the certificates for securing the kubelet service.
- Then provide the CA file using the `--client-ca-file=/path/to/ca.crt`

```sh
ExecStart=/usr/local/bin/kubelet \\
  ...
 --client-ca-file=/path/to/ca.crt \\
  ...
```
kubelet-config.yaml:
```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  x509:
    clientCAFile: /path/to/ca.crt
```
- Now that the certificate is configured here to access the API, you must provide the client certificate to authenticate to kubelet.
- `kubelet-cert.pem` (This is the client certificate that curl sends to kubelet) and `kubelet-key.pem` (This is the private key for that client certificate) are client certificate files used by curl to authenticate to the kubelet API
```sh
curl -sk https://localhost:10250/pods/ -key kubelet-key.pem -cert kubelet-cert.pem
```

API Server sometimes needs to connect to the kubelet on each node (API Server is as a Client): 
```sh
cat /etc/systemd/system/kube-apiserver.service

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
# ...
--kubelet-client-certificate=/path/to/apiserver-kubelet-client.crt \
--kubelet-client-key=/path/to/apiserver-kubelet-client.key \
# ...
```
Because this certificate is used by the **API Server as a client** when it talks to the kubelet.

> [!important]
- `--anonymous-auth=false` applies to port `10250`, not the read-only port `10255`
- For Port `10255` : Disable it by setting `readOnlyPort: 0` as If you do not set it to `0`, kubelet may expose the read-only port , Then anyone who can reach that node/port may be able to read endpoints like: `curl http://<node-ip>:10255/metrics` , `curl http://<node-ip>:10255/pods`

### Authorization ^autho
- The default authorization mode is `AlwaysAllow`, and this allows all access to the API.
```sh
ExecStart=/usr/local/bin/kubelet \\
# ...
  --authorization-mode=AlwaysAllow
# ...
```

- **Problem**: If someone has a valid certificate or bearer token and can reach kubelet port `10250`, kubelet may allow access to endpoints like: `/pods`, `/exec` This is dangerous because authorization is too weak.
- **Solve**: Change `AlwaysAllow` to `Webhook`: the kubelet makes a call to the API server to determine whether each request can be authorized or not. And depending on the result received from the API server, it either rejects or approves the request.
- Means the `kubelet` will **stop automatically allowing every request** and will instead ask the `kube-apiserver` if the request is allowed.

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
readOnlyPort: 0
```

> [!summary]

- 10250 = secure kubelet HTTPS port. `anonymous: false` + `Use CA certificate` + `authorization Webhook` = protect it
- 10255 = insecure read-only kubelet port. `readOnlyPort: 0` = disable it

`kubelet.service` : 
```sh
ExecStart=/usr/local/bin/kubelet \\
  ...
  --anonymous-auth=false \\
  --client-ca-file=/path/to/ca.crt \\
  --authorization-mode=Webhook
  --read-only-port=0
  ...
```

`kubelet-config.yaml`
```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  anonymous:
    enabled: false

  x509:
    clientCAFile: /path/to/ca.crt

authorization:
  mode: Webhook

readOnlyPort: 0
```
Meaning:
```text
anonymous.enabled: false
Disables anonymous access to kubelet secure port 10250.

clientCAFile: /path/to/ca.crt
Allows kubelet to verify client certificates.

authorization.mode: Webhook
Kubelet asks kube-apiserver whether the authenticated request is allowed.

readOnlyPort: 0
Disables the insecure read-only kubelet port 10255.
```

---

## 4. Cluster Upgrade
**Pre-Upgrade Notes**
- Upgrade one minor version at a time **(e.g., 1.10 → 1.11 → 1.12, never skip versions)**
- Kubernetes supports only the last 3 minor versions

[doc - Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
[doc - Changing The Kubernetes Package Repository](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/)


**Phase 1: Check upgrade plan**
`kubectl get nodes` Show Current Versions

How many nodes can host workloads in this cluster? 
`kubectl describe nodes  controlplane | grep -i taint`
`kubectl describe nodes  node01 | grep -i taint`

`kubeadm upgrade plan | grep -i remote` Show kubeadm latest version available

`kubeadm upgrade plan` Shows current versions, available versions, and what needs manual upgrade.

**Phase 2: Upgrade the Master Node**
1. Upgrade kubeadm on master
`apt-get upgrade -y kubeadm=1.12.0-00`

2. Apply the upgrade
`kubeadm upgrade plan v1.12.0`
`kubeadm upgrade apply v1.12.0`

3. Upgrade kubelet on master (kubeadm does NOT do this automatically)
`apt-get upgrade -y kubelet=1.12.0-00`

4. Restart kubelet
`systemctl restart kubelet`

5. Verify
`kubectl get nodes`

**While the master is upgrading, worker nodes keep serving traffic normally. Management functions (kubectl, controller-manager) are temporarily unavailable.**

**Phase 3: Upgrade Worker Nodes (one at a time)**
Run steps `1 and 6` from the `master node`, steps `2 to 5` on the `worker node`:

1. Drain the node **(run on master)**
`kubectl drain node-1 --force --ignore-daemonsets`

2. Upgrade kubeadm (run on the worker node)
`apt-get upgrade -y kubeadm=1.12.0-00`

3. Upgrade kubelet (run on the worker node)
`apt-get upgrade -y kubelet=1.12.0-00`

4. Update kubelet config (run on the worker node)
`kubeadm upgrade node config --kubelet-version v1.12.0`

5. Restart kubelet (run on the worker node)
`systemctl restart kubelet`

6. Uncordon the node **(run on master)**
`kubectl uncordon node-1`

**Repeat Phase 3 for each worker node.**

**Labs - Cluster Upgrade**
1. Upgrade The Master Node
```sh
kubectl drain controlplane --force --ignore-daemonsets

vim /etc/apt/sources.list.d/kubernetes.list
# Update the version in the URL to the next available minor release, i.e v1.35. 
# deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /

apt update
apt-cache madison kubeadm
# Based on the version information displayed by apt-cache madison, it indicates that for Kubernetes version 1.35.0, one of the available package versions is 1.35.0-1.1. Therefore, to install kubeadm for Kubernetes v1.35.0, use the following command:

apt-get install kubeadm=1.35.0-1.1

kubeadm upgrade plan v1.35.0
kubeadm upgrade apply v1.35.0

apt-get install kubelet=1.35.0-1.1

systemctl daemon-reload
systemctl restart kubelet

kubectl uncordon controlplane
kubectl get node
```

2. Upgrade The Worker Node
```sh
kubectl drain node01 --force --ignore-daemonsets

ssh node01
vim /etc/apt/sources.list.d/kubernetes.list
# add: deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /

apt update
apt-cache madison kubeadm

apt-get install kubeadm=1.35.0-1.1
kubeadm upgrade node

apt-get install kubelet=1.35.0-1.1

systemctl daemon-reload
systemctl restart kubelet

exit 

kubectl uncordon node01
```

---

## 5. Ingress
- Ingress is a Kubernetes object used to expose HTTP/HTTPS applications outside the cluster.
- It gives external users access to Services inside Kubernetes using URLs like: `https://app.example.com`

Ingress has two parts:
1. Ingress Resource
The Ingress Resource is the YAML object you create. It contains the routing rules.
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```
When traffic comes to `app.example.com/api`, send it to `api-service` on port `80`
> [!important]
> - Ingress Resource is only configuration.
> - It does not receive traffic by itself.

2. Ingress Controller
The Ingress Controller is the real running component that reads the Ingress Resource and applies the routing.
Examples of Ingress Controllers:
```text
NGINX Ingress Controller
Traefik
HAProxy Ingress
AWS ALB Controller
GCE Ingress Controller
Istio Ingress Gateway
```
The controller usually runs as Pods inside the cluster.
```sh
kubectl get pods -n ingress-nginx
```
**The controller receives external traffic and forwards it to the correct Service.**

>[!summary]
> - **Ingress Resource** = traffic rules / configuration file
> - **Ingress Controller** = actual router / reverse proxy that uses those rules
> - Like Nginx:
> 	- nginx.conf = Ingress Resource
> 	- nginx process = Ingress Controller

### Ingress Types
#### 1. Path-based Ingress
Routes traffic based on URL path.
- example.com/wear   → wear-service
- example.com/watch  → watch-service

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /wear
            pathType: Prefix
            backend:
              service:
                name: wear-service
                port:
                  number: 80

          - path: /watch
            pathType: Prefix
            backend:
              service:
                name: watch-service
                port:
                  number: 80
```
Two Services `wear-service` and `watch-service` can expose the same Service port `80` This is valid because each Service has its own ClusterIP.
#### 2. Host-based Ingress
Routes traffic based on domain/subdomain.
- wear.example.com   → wear-service
- watch.example.com  → watch-service

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: wear.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: wear-service
                port:
                  number: 80

    - host: watch.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: watch-service
                port:
                  number: 80
```

#### 3. Default Backend
Used when no rule matches.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: default-backend-ingress
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: default-service
      port:
        number: 80
```

---

## 6. Docker Service Configuration
- `dockerd` is the Docker daemon.
- It is the background process that actually manages Docker on your machine.
- When you run `docker run nginx` , the `docker` command is only the client.
- The client sends requests to `dockerd` , Then `dockerd` does the real work.
- **Simple architecture**: `docker CLI` => `dockerd` => `containerd` => `runc` => `Linux kernel`
- To show docker daemon logs run `dockerd --debug` 
- So when the Docker daemon starts, it listens on an internal Unix socket at the path `/var/run/docker.sock`
- A Unix socket is an inter-process communication mechanism that is used for communication between different processes on the same host.
- This means that the Docker daemon is only accessible within the same host, and the Docker CLI is configured to interact with the Docker daemon on this socket.
- To establish a connectivity from outside this host (there is another host with docker CLI installed on it and want to target the Docker host to run contianers) make the docker daemon listen on a **TCP interface** on the docker host by adding `--host`

```sh
dockerd --debug \
	--host=tcp://192.168.1.10:2375
```

- In this case, our host happens to have an interface at `192.168.1.10` and the standard port for Docker happens to be `2375`.
- Now the Docker daemon is accessible from outside of the Docker host using the IP `192.168.1.10` and the port `2375`
- The other host can now target the Docker daemon using the docker command: 
```sh
export DOCKER_HOST="tcp://192.168.1.10:2375"
# then run
docker ps 
```
 
- By default, The Docker daemon serves unencrypted traffic.

1. To enable **Encryption**
- In this Step , we will make an Encryption only not Authentication, you must first create **a pair of TLS certificates** and add the **TLS flag** to true as shown below

```sh
# We have the Docker Host - Server and Another Host with Docker Cli
# From the Docker Host:
dockerd --debug \
	--host=tcp://192.168.1.10:2376 \
	--tls=true \
	--tlscert=/var/docker/server.pem \
	--tlskey=/var/docker/serverkey.pem
	
# From Client host change the port from 2375 to 2376 
export DOCKER_HOST="tcp://192.168.1.10:2376"
# 2375 => Unencrypted
# 2376 => Encrypted

export DOCKER_TLS=true
# or use flag: --tls 
docker --tls ps
```

In The Docker host, Can use the config file: `/etc/docker/daemon. json`
```json
"debug": true,
"hosts": ["tcp://192.168.1.10:2376"],
"tls": true,

// TLS certificate and private key for the Docker daemon server
"tlscer": "/var/docker/server.pem",
"tlskey": "/var/docker/serverkey.pem"
```

- `--tlscert=/var/docker/server.pem`: This is the server certificate used by `dockerd`, It proves to Docker clients: `I am the real Docker daemon running on 192.168.1.10.`
- `--tlskey=/var/docker/serverkey.pem`: This is the private key for that server certificate. It proves that dockerd owns the certificate. This file must be kept secret.
- **Note**: `--tls` option alone does **not authenticate**, it only **enables encryption** and **anyone can still access the docker daemon**

2. To Enable **Authentication (certificate-based authentication)**
- Enables authentication: `"tlsverify": true`
- The CA Certificate `"tlscacert": "/var/docker/caserver.pem"` is used by the Docker daemon to verify client certificates.
- So our clients now need certificates signed by the CA and that's the only way now they can access the Docker daemon.
- So we generate certificates for our clients called `client.pem` and `client-key.pem` and share that securely with the client that wants to connect to the host Along with the certificate of the `CA`.

From Docker Host: `/etc/docker/daemon.json`
```json
{
"hosts": ["tcp://192.168.1.10:2376"]
"tls": true,
"tlscert": "/var/docker/server.pem",
"tlskey": "/var/docker/serverkey.pem",

// enables authentication:
"tlsverify": true,
// 1. Docker client verifies the Docker server.
// 2. Docker server also verifies the Docker client.

// This is the CA certificate used by the Docker daemon to verify client certificates.
"tlscacert": "/var/docker/caserver.pem",
// Only Docker clients with certificates signed by this CA are trusted.

}
```
From Client host:
```sh
export DOCKER_TLS_VERIFY=true

docker \
  --host=tcp://192.168.1.10:2376 \
  --tlsverify \
  --tlscacert=/path/to/ca.pem \
  --tlscert=/path/to/client.pem \
  --tlskey=/path/to/client-key.pem \
  ps

# or drop the certificates in ~/.docker directory

# ca.pem: The CA certificate used by the Docker client to verify the Docker server certificate. 
# client.pem: The client certificate sent by Docker client to authenticate to the Docker daemon.
# client-key.pem: The private key for the client certificate.
```

> [!summary]
> Docker client
>    ↓
> connects to tcp://192.168.1.10:2376
>    ↓
> dockerd sends server.pem
>    ↓
> client verifies server.pem using ca.pem
>    ↓
> client sends client.pem
>    ↓
> dockerd verifies client.pem using ca.pem
>    ↓
> if both are valid, connection is allowed
>    ↓
> docker ps runs remotely

---

## 7. Securing Node Metadata
- Node metadata means the descriptive information attached to a Node object like: `name` , `labels`, `annotations`, `system info`
- We secure node metadata because it can leak cluster information and cloud credentials, which attackers can use for privilege escalation and lateral movement.

**1. Prevent Unauthorized Access**
Identify vulnerabilities in this version of kubelet
```sh
kubectl get nodes -o jsonpath='{.items[*].status.nodeInfo.kubeletVersion}'
# Example output:
# 1.26.0
```
Risk:
If the attacker knows the kubelet version, they can search for known vulnerabilities affecting that specific version and try to exploit them.

**2. Protect Against Misconfiguration**
Example misconfiguration, An inexperienced user accidentally removes a critical taint from a production node:
```sh
kubectl taint nodes node-1 key=value:NoSchedule-
# Example output:
# node/node-1 untainted
```
Risk:
Removing important taints can break workload isolation and allow the wrong Pods to run on sensitive production nodes.

**3. Maintain Privacy**
Attacker lists internal node IP addresses
```sh
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
# 192.168.1.10 192.168.1.11 192.168.1.12
```
Risk:
Internal node IPs reveal the cluster network layout and can help attackers targets attacks.

**4. Ensure Compliance**
Unauthorized access to node kernel versions
```sh
kubectl get nodes -o jsonpath='{.items[*].status.nodeInfo.kernelVersion}'
# 5.4.0-1041-aws 4.15.0-142-generic 5.8.0-53-generic
```
Risk:
Kernel versions can reveal sensitive infrastructure details and may expose whether nodes are running outdated or vulnerable kernels.

These are controls used to secure Kubernetes node metadata: 
- **RBAC**: controls who can access and modify node metadata
- **Node isolation**: reserves specific nodes for designated workloads, ensuring that less critical or unauthorized applications are prevented from running on them.
- **Audit logs**: track who accessed or modified node metadata and when.
- **Regular updates and patches**: to ensure that vulnerabilities in the metadata or infrastructure aren’t exploited.

**Solution: Block pods from reaching cloud metadata endpoint**: 
- **Note**: `169.254.169.254` is not a public IP. It is a link-local address, reachable only from the local machine/network namespace, not routed on the public internet.
- To prevent any Pod in a namespace from reaching the **IP address of the metadata server**, set up a network policy that allows egress traffic to all IP addresses except `169.254.169.254`
- For metadata endpoints, we block IPs like `169.254.169.254` as Most cloud metadata services use `169.254.169.254` , so we need: 
	- Allow egress to all IPv4 addresses
	- except `169.254.169.254/32`
	
```sh
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-metadata-server
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 169.254.169.254/32
```
Run a test pod in the namespace:
```sh
kubectl run test-curl \
  -n production \
  --image=curlimages/curl \
  --rm -it -- sh
  
# Inside the pod:
curl -m 3 http://169.254.169.254
# Expected result:
# Connection timed out , Could not connect
```

---

## 8. Auditing
Auditing is essential for tracking who did what, when, and where within a Kubernetes cluster.
- `Audit Policy` is a configuration file used by the kube-apiserver to decide:
	- Which Kubernetes API requests should be logged?
	- How much detail should be logged?
	- Which requests should be ignored?

**Audit levels**
Kubernetes audit policy has different logging levels:
1. **None**: No events are logged.
2. **Metadata**: Logs only request metadata (user, timestamp, resource, verb) without request/response bodies.
3. **Request**: Logs both event metadata and the request body, but excludes the response body.
4. **Request Response**: Logs event metadata, the request body, and the response body.


**1. None Audit Policy**:
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: None
    verbs: ["get", "list"]
    resources:
      - group: ""
        resources: ["pods"]
```
- `level: None` means: Do not log matching requests.
- Any request matching this rule `get` , `list` will be ignored by the audit log.

**2. Metadata Policy**: 
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
    verbs: ["get", "list", "create", "delete", "update"]
    resources:
      - group: ""
        resources: ["pods"]
```
Audit log event output:
```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "timestamp": "2024-10-11T12:34:56Z",
  "user": {
    "username": "system:serviceaccount:kube-system:default",
    "groups": ["system:serviceaccounts", "system:serviceaccounts:kube-system"]
  },
  "verb": "get",
  "namespace": "default",
  "resource": "pods",
  "stage": "ResponseComplete",
  "objectRef": {
    "resource": "pods",
    "namespace": "default",
    "name": "example-pod"
  }
}
```

**3. Request Policy**
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Request
    verbs: ["create", "delete", "update"]
    resources:
      - group: ""
        resources: ["pods"]
```
Audit Log Event Output: 
```json
{
// Metadata
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Request",
  "timestamp": "2024-10-11T12:34:56Z",
  "user": {
    "username": "system:serviceaccount:kube-system:default",
    "groups": ["system:serviceaccounts", "system:serviceaccounts:kube-system"]
  },
  "verb": "create",
  "namespace": "default",
  "resource": "pods",
  "stage": "ResponseComplete",
  "objectRef": {
    "resource": "pods",
    "namespace": "default",
    "name": "example-pod"
  },
  
  // Request Object
  "requestObject": {
    "metadata": {
      "name": "example-pod",
      "namespace": "default"
    },
    "spec": {
      "containers": [
        {
          "name": "example-container",
          "image": "nginx"
        }
      ]
    }
  }
}
```

**4. RequestResponse**
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: RequestResponse
    verbs: ["create", "delete", "update"]
    resources:
      - group: ""
        resources: ["pods"]
```
Audit Log Event Output:
```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "RequestResponse",
  "timestamp": "2024-10-11T12:34:56Z",
  "user": {
    "username": "system:serviceaccount:kube-system:default",
    "groups": ["system:serviceaccounts", "system:serviceaccounts:kube-system"]
  },
  "verb": "create",
  "namespace": "default",
  "resource": "pods",
  "stage": "ResponseComplete",
  "objectRef": {
    "resource": "pods",
    "namespace": "default",
    "name": "example-pod"
  },
  "requestObject": {
    "metadata": {
      "name": "example-pod",
      "namespace": "default"
    },
    "spec": {
      "containers": [
        {
          "name": "example-container",
          "image": "nginx"
        }
      ]
    }
  },
  "responseObject": {
    "metadata": {
      "name": "example-pod",
      "namespace": "default",
      "uid": "12345-67890-abcd",
      "creationTimestamp": "2024-10-11T12:34:56Z"
    },
    "status": {
      "phase": "Pending"
    }
  }
}
```

---
