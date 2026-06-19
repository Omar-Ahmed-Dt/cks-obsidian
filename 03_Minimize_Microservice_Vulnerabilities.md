---
tags:
  - kubernetes
  - cks
---
# Security Contexts
You can run a container with a designated user or add a capability using the following commands: 
```sh
docker run --user=1001 ubuntu sleep 3600
docker run --cap-add MAC_ADMIN ubuntu
```
In Kubernetes, Capabilities are only supported at the container level and not at the POD level:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  # runAsUser config is a Pod Level config
  securityContext:
    runAsUser: 1000
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 1000
		# Container Level
        capabilities:
          add: ["MAC_ADMIN"]
```

---

# Admission Controllers
- Admission Controllers are security/control plugins that run inside the kube-apiserver after authentication and authorization, but before the object is saved in etcd.
- Admission controllers in Kubernetes, which **validate**, **mutate** before they are persisted.

```text
kubectl apply
   |
   v
API Server
   |
   |-- 1. Authentication: who are you?
   |-- 2. Authorization: are you allowed?
   |-- 3. Admission Controllers: is this request acceptable or should it be changed?
   |
   v
etcd
```
## Request Lifecycle in Kubernetes

When a request is sent to the API server, it undergoes several critical steps:

1. **Authentication**
   The API server authenticates the request. For instance, when using kubectl, the KubeConfig file supplies the necessary certificates.

   ```sh
   cat ~/.kube/config
   ```

   ```yaml
   apiVersion: v1
   clusters:
   - cluster:
       certificate-authority-data: LS0tLS1CRUdJTiBDRVUx...
   ```

2. **Authorization**
   After authentication, the request is authorized. Kubernetes uses role-based access control (RBAC) to determine if the user has permission to perform the requested operation. For example, a role allowing the manipulation of pods might be defined as:

   ```yaml theme={null}
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     name: developer
   rules:
   - apiGroups: [""]
     resources: ["pods"]
     verbs: ["list", "get", "create", "update", "delete"]
   ```

   In below case, the developer is restricted to creating pods named either "blue" or "orange."

   ```yaml theme={null}
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     name: developer
   rules:
   - apiGroups: [""]
     resources: ["pods"]
     verbs: ["create"]
     resourceNames: ["blue", "orange"]
   ```


## The Role of Admission Controllers

Beyond basic authentication and authorization, there are scenarios that require additional validations or modifications requests. Consider a pod creation request, where you might want to:
- Ensure that images are only pulled from an approved internal registry.
- Enforce that the image tag is not set to "latest."
- Reject requests if the container runs as the root user.
- Modify the container’s security context or enforce specific metadata labels.

Take this pod manifest as an example:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu:latest
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 0
        capabilities:
          add: ["MAC_ADMIN"]
```

**RBAC does not cover these complex validations or modifications**. That is where admission controllers come into play; they provide an additional security layer by examining, modifying, or rejecting API requests before they reach etcd.

<Frame>
  ![The image illustrates the Kubernetes process flow: Kubectl command, authentication, authorization, admission controllers, and finally, pod creation.](https://kodekloud.com/kk-media/image/upload/v1752871631/notes-assets/images/Certified-Kubernetes-Security-Specialist-CKS-Admission-Controllers/frame_210.jpg)
</Frame>

### Built-In Admission Controllers

Kubernetes includes several pre-built admission controllers, such as:

* **Always Pull Images:** Ensures that images are pulled on each pod creation.
* **Default Storage Class:** Automatically assigns a default storage class to persistent volume claims if none is specified.
* **Event Rate Limit:** Restricts the number of requests processed by the API server concurrently.
* **Namespace Exists:** Verifies that the specified namespace exists, rejecting requests for non-existent namespaces.

## Namespace-Related Admission Controllers

### Namespace Exists Admission Controller

If you attempt to create a pod in a non-existent namespace, the namespace exists admission controller will reject the request. For example:

```bash theme={null}
kubectl run nginx --image nginx --namespace blue
```

The flow is as follows:

1. The API server authenticates and authorizes the request.
2. The namespace exists admission controller verifies if the "blue" namespace is available.
3. Since the namespace does not exist, the request is rejected.

### Namespace Auto-Provision Admission Controller

An alternate admission controller, the namespace auto-provision admission controller, can automatically create a namespace if it does not exist. Note that this controller is not enabled by default. Without auto-provisioning, running the command:

```bash theme={null}
kubectl run nginx --image nginx --namespace blue
```

results in:

```bash theme={null}
Error from server (NotFound): namespaces "blue" not found
```

To see which admission controllers are enabled by default, run:

```bash theme={null}
kube-apiserver -h | grep enable-admission-plugins
```

If your cluster uses a kubeadm-based setup, execute this command within the kube-apiserver control plane pod:

```bash theme={null}
kubectl exec kube-apiserver-controlplane -n kube-system -- kube-apiserver -h | grep enable-admission-plugins
```

## Configuring Admission Controllers

### Enabling Admission Controllers

To enable additional admission controllers, update the `--enable-admission-plugins` flag on the kube-apiserver. In a kubeadm-based setup, this update is performed in the kube-apiserver manifest file. For example, you might configure the API server service as follows:

```bash theme={null}
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --enable-swagger-ui=true \\
  --etcd-servers=https://127.0.0.1:2379 \\
  --event-ttl=1h \\
  --runtime-config=api/all \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --v=2 \\
  # Enable namespaceAutoProvision Admission Controller: 
  --enable-admission-plugins=NodeRestriction,NamespaceAutoProvision
```

When the API server runs as a pod in a kubeadm-based setup, the manifest might look like this:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
    - name: kube-apiserver
      image: k8s.gcr.io/kube-apiserver-amd64:v1.11.3
      command:
        - kube-apiserver
        - --authorization-mode=Node,RBAC
        - --advertise-address=172.17.0.107
        - --allow-privileged=true
        - --enable-bootstrap-token-auth=true
		# Enable namespaceAutoProvision Admission Controller: 
        - --enable-admission-plugins=NodeRestriction,NamespaceAutoProvision
```

To disable specific admission controller plugins, leverage the `--disable-admission-plugins` flag in a similar way.

### Testing Auto-Provisioning

After enabling the desired admission controllers, a pod creation request in a non-existent namespace behaves differently. With the namespace auto-provision controller enabled, executing:

```bash theme={null}
kubectl run nginx --image nginx --namespace blue
```

will successfully create the pod. Upon listing namespaces with:

```bash theme={null}
kubectl get namespaces
```

you should observe that the "blue" namespace has been automatically created:

```plaintext theme={null}
NAME         STATUS   AGE
blue         Active   3m
default      Active   23m
kube-public  Active   24m
kube-system  Active   24m
```

## Deprecation Notice

Note that the **namespace auto-provision** and **namespace existence admission** controllers have been **deprecated and replaced by the namespace lifecycle admission controller**. The namespace lifecycle admission controller now ensures that requests targeting non-existent namespaces are rejected, while also safeguarding critical namespaces (such as default, kube-system, and kube-public) from deletion.

---

# Validating and Mutating Admission Controllers

Types of Admission Controllers in Kubernetes:
* **Validating Admission Controllers:** These controllers check incoming requests and either **allow or deny** them (Without any Modification) based on predefined rules.
* **Mutating Admission Controllers:** These controllers **modify** requests by adjusting the object before it is persisted to the cluster.

## Validating Admission Controllers
- One example of a validating admission controller is the namespace existence (or namespace lifecycle) controller, which ensures that a namespace exists before allowing a request. If the namespace does not exist, the request is rejected.

## Mutating admission controllers 
- it is For modify the request while validating admission controllers only verify the request against set policies.
- one example of a mutating admission controller is the default storage class admission controller, which is enabled by default. Consider the following scenario: when you submit a request to create a `PersistentVolumeClaim (PVC)` without specifying a `storage class`, the built-in admission controller intervenes by modifying the request to include the `preconfigured default storage class`

Example: PVC Request without a Storage Class: 
```yaml theme={null}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

This PVC creation request passes through several stages: authentication, authorization, and finally, the admission controllers. The default storage class controller detects the missing storage class and automatically adds it to the request. The resulting PVC appears as follows:
```yaml theme={null}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: default
```

To inspect the PVC, use:

```bash theme={null}
kubectl describe pvc myclaim
```

You might see an output similar to:

```text theme={null}
Name:          myclaim
Namespace:     default
StorageClass:  default ##############################
Status:        Pending
Volume:        <none>
Labels:        <none>
Annotations:   <none>
```

- Typically, **mutating controllers run before validating controllers** so that any modifications are validated.
- For example, a namespace auto-provisioning controller (a mutating controller) can create missing namespaces before the validating "namespace exists" controller runs. If the order were reversed, the validating controller might reject requests for non-existent namespaces, preventing auto-provisioning from occurring.

If any **admission controller** in the processing chain rejects a request, the entire operation fails and an error message is returned to the user.

## Extending Admission Controllers with Webhooks

In addition to built-in admission controllers, Kubernetes allows you to implement custom logic via **two types of external webhooks**:
* **Mutating Admission Webhook**
* **Validating Admission Webhook**

These webhooks let you direct admission review requests to a custom server—either within or outside your cluster. **After the built-in admission controllers process a request, it is forwarded to the webhook**. The webhook server receives an admission review object in JSON format containing details such as the user, requested operation, and the object involved.

### Admission Review JSON Object Example

Below is an example of the JSON object sent to a webhook server:

```json theme={null}
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "request": {
    "uid": "705ab4f5-6393-11e8-b7cc-4201aa800002",
    "kind": {"group": "autoscaling", "version": "v1", "kind": "Scale"},
    "resource": {"group": "apps", "version": "v1", "resource": "deployments"},
    "subResource": "scale",
    "requestKind": {"group": "autoscaling", "version": "v1", "kind": "Scale"},
    "requestResource": {"group": "apps", "version": "v1", "resource": "deployments"}
  }
}
```

The webhook server processes this and responds with an object indicating if the request is allowed. For instance, an approval response might be:

```json theme={null}
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "request": {
    "uid": "705ab4f5-6393-11e8-b7cc-42010aa80002",
    "kind": {"group": "autoscaling", "version": "v1", "kind": "Scale"},
    "resource": {"group": "apps", "version": "v1", "resource": "deployments"},
    "subResource": "scale",
    "requestKind": {"group": "autoscaling", "version": "v1", "kind": "Scale"},
    "requestResource": {"group": "apps", "version": "v1", "resource": "deployments"}
  },
  "response": {
    "uid": "value_from_request.uid",
    "allowed": true
  }
}
```

If the "allowed" field is false, the webhook will cause the API server to reject the request.

## Deploying an Admission Webhook Server

To utilize a custom admission controller, you must deploy your own webhook server, which contains the custom logic for mutation and/or validation. The server can be developed using any programming language that supports HTTPS (TLS is required for secure communication with the Kubernetes API server).

### Example: Go Webhook Server

A sample admission webhook server written in Go:

```go theme={null}
package main

import (
    "encoding/json"
    "flag"
    "fmt"
    "io/ioutil"
    "net/http"

    "k8s.io/api/admission/v1beta1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/klog"
)

// toAdmissionResponse is a helper function to create an AdmissionResponse with an embedded error.
func toAdmissionResponse(err error) v1beta1.AdmissionResponse {
    return v1beta1.AdmissionResponse{
        Result: &metav1.Status{
            Message: err.Error(),
        },
    }
}

// admitFunc defines the function signature used for validators and mutators.
type admitFunc func(v1beta1.AdmissionReview) v1beta1.AdmissionResponse

// serve handles the HTTP portion of a request prior to passing it to an admit function.
func serve(w http.ResponseWriter, r *http.Request, admit admitFunc) {
    var body []byte
    if r.Body != nil {
        if data, err := ioutil.ReadAll(r.Body); err == nil {
            body = data
        }
    }
    // Further processing would continue here...
}
```

Although this example is written in Go, you can build your webhook server in any language that accommodates HTTPS and JSON-based API communications.

### Example: Python Webhook Server

The following pseudocode demonstrates a simple webhook server in Python using Flask. It includes **two routes - two calls**: one for validation and another for mutation.

```python theme={null}
from flask import Flask, request, jsonify
import base64

app = Flask(__name__)

@app.route("/validate", methods=["POST"])
def validate():
    object_name = request.json["request"]["object"]["metadata"]["name"]
    user_name = request.json["request"]["userInfo"]["name"]
    status = True
    message = ""
    if object_name == user_name:
        message = "You can't create objects with your own name"
        status = False
    return jsonify(
        {
            "response": {
                "allowed": status,
                "uid": request.json["request"]["uid"],
                "status": {"message": message},
            }
        }
    )

@app.route("/mutate", methods=["POST"])
def mutate():
    user_name = request.json["request"]["userInfo"]["name"]
    patch = [{"op": "add", "path": "/metadata/labels/users", "value": user_name}]
    # Encode the patch using base64
    patch_encoded = base64.b64encode(str(patch).encode()).decode()
    return jsonify(
        {
            "response": {
                "allowed": True,
                "uid": request.json["request"]["uid"],
                "patch": patch_encoded,
                "patchType": "JSONPatch",
            }
        }
    )

if __name__ == "__main__":
    app.run(debug=True, port=443)
```

In this Python example, the validation endpoint rejects requests where the object's name matches the user name, while the mutation endpoint adds a label with the username using a JSON patch.

## Configuring the Webhook in Kubernetes
After deploying your webhook server, configure your Kubernetes cluster to use it by creating a **webhook configuration object**. Below is an example of a `ValidatingWebhookConfiguration`:
```yaml theme={null}
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: "pod-policy.example.com"
webhooks:
	# A ValidatingWebhookConfiguration can contain one or more webhooks.
	# The webhook name should be unique and usually domain-based.
  - name: "pod-policy.example.com"
    clientConfig: # This tells the API server where to send the admission request.
      service: # This means the webhook backend is running behind this Kubernetes Service
        namespace: "webhook-namespace"
        name: "webhook-service"
      caBundle: "CiOtLS0tQk......tLS0K"
	  # This is the CA certificate used by the Kubernetes API server to trust the webhook service TLS certificate.
	  # Because admission webhooks use HTTPS, the API server must verify the webhook certificate.
    rules:
	    # when someone tries to create a Pod, Kubernetes API Server will call your webhook service first. Your service checks the Pod and replies:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
        scope: "Namespaced"
```

In this configuration:

* The webhook is triggered during pod creation.
* TLS is used for secure communication, as indicated by the `caBundle`.
* The API server references the webhook service by its name and namespace when deployed within the cluster.

For mutating webhooks, a similar configuration is created with `kind: MutatingWebhookConfiguration`.
Once applied, every time a pod is created (or another resource event specified in your rules), the API server calls your webhook server. Depending on whether the response indicates approval or rejection, the API server will allow or reject the request.

---

# Pod Security Policies
Below is a sample pod definition that creates a pod running an Ubuntu container. The configuration includes permissive settings often unsuitable for production environments:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        privileged: True
        runAsUser: 0
        capabilities:
          add: ["CAP_SYS_BOOT"]
  volumes:
    - name: data-volume
      hostPath:
        path: /data
        type: Directory
```

In this pod:
- The `privileged` flag is set to `True`, granting container processes elevated privileges.
	- Normal container = isolated and limited
	- Privileged container = almost like root on the Kubernetes node
	
- The container runs as root `runAsUser: 0`
- It adds specific capabilities like `CAP_SYS_BOOT`
- A `HostPath` volume is used, exposing part of the host’s filesystem.

**Difference between** `runAsUser: 0` and `privileged: True`
- `runAsUser: 0`: Run as root user inside the container , But root inside a normal container is still limited.
- `privileged: True`: Give the container host-level privileges

These settings can introduce security vulnerabilities. To safeguard your cluster, it is important to enforce policies that restrict such configurations.


## Pod Security Admission and Pod Security Standards - PSA and PSS
- Pod Security Standards (PSS) = the security rules.
- Pod Security Admission (PSA) = the Kubernetes built-in admission controller that enforces those rules on Pods.

Think of it like this:
- PSS = policy definitions
- PSA = the controller that applies/enforces them

**PSA** operates as an Admission Controller and is enabled by default in Kubernetes clusters. To verify its activation, run the following command to display the enabled admission plugins in the kube-apiserver:
```sh
kubectl exec -n kube-system kube-apiserver-controlplane -it -- kube-apiserver -h | grep enable-admission-plugins
# ...
# PodSecurity
# ...
```
PSA is configured at a namespace level, and you do that by applying a label to the namespace:
```sh
kubectl label namespace <namespace> pod-security.kubernetes.io/<mode>=<pod security standard - PSS>
```
- **mode**: what Kubernetes should do when a Pod violates the policy
- **PSS or level**: which security policy Kubernetes should compare the Pod against - like a policy profile

A mode determines the action taken when a policy violation occurs. The available modes are:
- **Enforce**: Rejects pod creation requests that do not comply with the specified policy - Rejects bad Pods
- **Warn**: Permits pod creation and issues a warning to the user regarding the policy violation - Allows Pod but shows warning - shows a warning to the user running kubectl
- **Audit**: Allows pod creation but logs the policy violation in the audit logs - Allows Pod but records violation in audit logs - writes the violation into Kubernetes audit logs

**PSS** - Pod Security Standards are predefined security levels for Pods: 
- **Privileged**: Almost no restrictions - Unrestricted policy
- **Baseline**: Blocks common dangerous settings - Minimally restrictive policy - block common privilege-escalation / host-access risks
- **Restricted**: Strongest built-in hardening - Heavily restricted policy

```sh
kubectl label namespace dev \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```
Meaning:
- enforce baseline  -> block Pods that violate baseline
- warn restricted    -> allow the Pod, but show warning if it violates restricted
- audit restricted   -> allow the Pod, but write audit log if it violates restricted

**Examples**
1. Pod blocked by `enforce=baseline`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: dev
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        privileged: true
```

```sh
kubectl apply -f privileged-pod.yaml
```
Expected result:
```text
Error from server (Forbidden): pods "privileged-pod" is forbidden:
violates PodSecurity "baseline:latest": privileged
```
Because `privileged: true` violates `baseline`

2. Pod allowed, but warning because it violates `restricted`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: normal-nginx
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx
```
Expected result:
```text
Warning: would violate PodSecurity "restricted:latest":
allowPrivilegeEscalation != false, unrestricted capabilities,
runAsNonRoot != true, seccompProfile
pod/normal-nginx created
```
- Why was it created? Because this Pod does not violate baseline.
- I will allow it now, but this Pod is not secure enough for restricted.

> [!summary]
> - `privileged: true` Pod => Blocked
> - normal nginx Pod => Created with warning
> - hardened Pod => Created without warning

> [!note]
> - PSA and PSS are good for standard Pod security, but they are limited because the rules are predefined.
> - Policy-as-Code - PAC tools allow you to write custom security rules: `Kyverno`, `OPA/Gatekeeper`, `Open Policy Agent`, `jsPolicy`

---

# Open Policy Agent OPA
- We have a web service using a simple Flask application.
- User is authenticated by the web server via username and password or certificates. OPA decides what the authenticated user is allowed to do.
- OPA is for authentication

we first demonstrate a basic Python-based Flask application without any authorization. Next, we incorporate basic authorization within Flask. Finally, we integrate OPA to provide a robust and flexible authorization solution: 
**A Simple Flask Application Without Authorization**
```python
@app.route('/home')
def hello_world():
    return 'Welcome Home!', 200
```
There is no authorization in place, meaning the endpoint is accessible to anyone.

**Adding Basic Authorization in Flask**
To add a simple layer of authorization, we check if the user is “john”. In this example, the username is provided as a URL parameter:
```python
@app.route('/home')
def hello_world():
    user = request.args.get("user")
    if user != "john":
        return 'Unauthorized!', 401
    return 'Welcome Home!', 200
```

## Deploying the OPA Server
With OPA, you define policies in a single location that all services can query via an API to determine access permissions
```sh
curl -L -o opa https://github.com/open-policy-agent/opa/releases/download/v0.11.0/opa_linux_amd64
chmod 755 ./opa
./opa run -s
# {"addrs":["8181"],"insecure_addr":"","level":"info","msg":"First line of log stream.","time":"2021-03-18T20:25:38+08:00"}
```
Starting the OPA server using the `-s` flag. By default, OPA listens on port `8181` and has an open API without built-in authentication or authorization.

### Defining an Authorization Policy with Rego
OPA policies are authoredin Rego and stored in files with the `.rego` extension. Below is an example policy that permits access to the `/home` endpoint only if the user is `john`:
```text
package httpapi.authz

# HTTP API request
import input

default allow = false

allow {
    input.path == "home"
    input.user == "john"
}
```
To load this policy into OPA, use a PUT request:
```sh
curl -X PUT --data-binary @example.rego http://localhost:8181/v1/policies/example1
```
To list all existing policies, run:
```sh
curl http://localhost:8181/v1/policies
```

### Integrating OPA with the Python Application
Instead of embedding the authorization check within your application code, you can now delegate it to the OPA server. The updated Flask application builds an input dictionary and sends it as JSON to the OPA API. The appropriate query endpoint for our policy package, `httpapi.authz`, is `/v1/data/httpapi/authz`: 
```python
@app.route('/home')
def hello_world():
    user = request.args.get("user")
    input_dict = {
        "input": {
            "user": user,
            "path": "home"
        }
    }
    rsp = requests.post("http://127.0.0.1:8181/v1/data/httpapi/authz", json=input_dict)
    if not rsp.json()["result"]["allow"]:
        return 'Unauthorized!', 401
    return 'Welcome Home!', 200
```
- With this implementation, the Flask application offloads the authorization decision to OPA.
- [For writing and testing policies](https://play.openpolicyagent.org/)

For instance, given the input:
```json
{
    "user": "john",
    "path": "home"
}
```
The policy will return:
```json
{
    "allow": true
}
```

OPA includes a built-in testing framework that allows you to run tests using Rego test files. Below is an example test file for an authorization policy:
```text
package authz

test_post_allowed {
    allow with input as {"path": ["users"], "method": "POST"}
}

test_get_anonymous_denied {
    not allow with input as {"path": ["users"], "method": "GET"}
}

test_get_user_allowed {
    allow with input as {"path": ["users", "bob"], "method": "GET", "user_id": "bob"}
}

test_get_another_user_denied {
    not allow with input as {"path": ["users", "bob"], "method": "GET", "user_id": "alice"}
}
```
Execute the tests with the following command:
```sh
$ opa test -v
# data.authz.test_post_allowed: PASS (1.417µs)
# data.authz.test_get_anonymous_denied: PASS (426ns)
# data.authz.test_get_user_allowed: PASS (367ns)
# data.authz.test_get_another_user_denied: PASS (320ns)
# -----------------------------------------------------------
# PASS: 4/4
```

## OPA in Kubernetes
- We will explore integrating OPA with Kubernetes using the Gatekeeper approach for enhanced policy enforcement and governance.
- This method leverages the OPA Constraint Framework alongside Kubernetes admission controllers for enhanced policy enforcement and governance
- With the Gatekeeper approach, the admission controller collaborates with the OPA Constraint Framework by using CRD-based (Custom Resource Definition) policies. This facilitates easier policy sharing and builds trust across your Kubernetes environment.

**Deploying OPA Gatekeeper**
```sh
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.14.0/deploy/gatekeeper
kubectl get all -n gatekeeper-system
```

### Implementing Label Validation with Rego
- The OPA Constraint Framework allows you to declare policies that specify required conditions, enforce those conditions at the appropriate locations, and define the checks to be performed.
- For example, if you want all objects in a specific namespace (e.g., "example") to include a "billing" label, the framework will enforce this rule via the Kubernetes admission controller.
- When a pod creation request is submitted, the admission controller follows these steps:
	- Retrieve the labels from the pod.
	- Verify if the required label (e.g., "billing") is present.
	- Return an error if the label is missing.

```text
package systemrequiredlabels

import data.lib.helpers

violation["msg": msg, "details": {"missing_labels": missing}} {
    provided := {label | input.request.object.metadata.labels[label]}
    required := {label | label == ["billing"]}
    missing = required - provided
    count(missing) > 0
    msg = sprintf("you must provide labels: %v", [missing])
}
```
In these examples:
- The `provided` variable extracts labels from the incoming pod object.
- The `required` set is fixed to include “billing”.
- The `missing` variable determines any labels from the `required` set that are absent.
- If any required labels are missing `count(missing) > 0`, an error message is generated.

To support more dynamic scenarios—such as enforcing different labels based on the namespace—you can create a `Constraint Template`. **This enables you to pass the required label as a parameter instead of hardcoding it**:
- The goal from below example is:
	- Namespace `expensive`   => all objects must have `label: billing`
	- Namespace `engineering` => all objects must have `label: tech`
	
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: systemrequiredlabels
spec:
  crd:
    spec:
      names:
        kind: SystemRequiredLabel
      validation:
        # Schema for the 'parameters' field goes here
  targets:
    - target: admission.k8s.gatekeeper.sh
	# targets tells Gatekeeper where this policy should run.
	# For Kubernetes admission control, the target is: admission.k8s.gatekeeper.sh
      rego: |
        package systemrequiredlabels

        import data.lib.helpers

        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
		  provided := {label | input.review.object.metadata.labels[label]}
		  # provided => This collects labels already existing on the object.
          # Use the parameter passed in the constraint instead of hardcoding: 
          required := {label | label == input.parameters.labels[_]}
		  # required => This reads labels from the constraint.
          missing = required - provided
          count(missing) > 0
          msg = sprintf("you must provide labels: %v", [missing])
        }
```
**What "target: admission.k8s.gatekeeper.sh" means here**:
this target means: Use this policy to validate Kubernetes objects using **Gatekeeper Admission** before they are saved.
```text
kubectl apply -f pod.yaml
        |
        v
kube-apiserver
        |
        v
Gatekeeper validating webhook
        |
        v
Rego policy under target: admission.k8s.gatekeeper.sh
        |
        v
allow or reject
```

**Rego Logic**: 
- Get labels from the Kubernetes object.
- Get required labels from the constraint parameters.
- Compare them: `required = {"billing"}` , `provided = {"app", "env"}` => **missing = {"billing"}**
- If required labels are missing, return violation.

```yaml
# For namespace expensive, every matched object must have label billing.

apiVersion: constraints.gatekeeper.sh/v1beta1
kind: SystemRequiredLabel
metadata:
  name: require-billing-label
spec:
  match:
    namespaces: ["expensive"]
  parameters:
    labels: ["billing"]
```

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: SystemRequiredLabel
metadata:
  name: require-tech-label
spec:
  match:
    namespaces: ["engineering"]
  parameters:
    labels: ["tech"]
```
These constraints dynamically pass the required labels via the `input.parameters` object in Rego based on the namespace.

**Example Files**
`requiredlabels-template.yaml`: 
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: systemrequiredlabels
spec:
  crd:
    spec:
      names:
        kind: SystemRequiredLabel
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
          required:
            - labels

  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package systemrequiredlabels

        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}

          missing := required - provided
          count(missing) > 0

          msg := sprintf("you must provide labels: %v", [missing])
        }
```

`require-label-billing.yaml`: 
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: SystemRequiredLabel
metadata:
  name: require-billing-label
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - expensive

  parameters:
    labels:
      - billing
```

```sh
kubectl apply -f requiredlabels-template.yaml
kubectl apply -f require-label-billing.yaml
# Output: Constraint require-billing-label created
```

**Real example**
To reject any Pod with:
```yaml
securityContext:
  privileged: true
```
using OPA/Gatekeeper, you need:
```text
1. ConstraintTemplate  -> contains the Rego policy logic
2. Constraint          -> applies the policy to Pods
3. Test Pods           -> one bad, one good
```

`01-deny-privileged-template.yaml`: 
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sdenyprivilegedcontainer
spec:
  crd:
    spec:
      names:
        kind: K8sDenyPrivilegedContainer
      validation:
        openAPIV3Schema:
          type: object
          properties: {}

  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdenyprivilegedcontainer

        violation[{"msg": msg, "details": {"container": container.name}}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true
          msg := sprintf("Privileged container is not allowed: %v", [container.name])
        }

        violation[{"msg": msg, "details": {"initContainer": container.name}}] {
          container := input.review.object.spec.initContainers[_]
          container.securityContext.privileged == true
          msg := sprintf("Privileged initContainer is not allowed: %v", [container.name])
        }

        violation[{"msg": msg, "details": {"ephemeralContainer": container.name}}] {
          container := input.review.object.spec.ephemeralContainers[_]
          container.securityContext.privileged == true
          msg := sprintf("Privileged ephemeralContainer is not allowed: %v", [container.name])
        }
```
This policy checks:
```text
spec.containers
spec.initContainers
spec.ephemeralContainers
```
If any container has: `privileged: true` => Gatekeeper returns a violation and rejects the request.

`02-deny-privileged-constraint.yaml`: 
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDenyPrivilegedContainer
metadata:
  name: deny-privileged-containers
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```
This applies the policy to all Pods in the cluster.

For testing only in one namespace, use this instead:
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDenyPrivilegedContainer
metadata:
  name: deny-privileged-containers
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - policy-demo
```

```sh
kubectl apply -f 01-deny-privileged-template.yaml
kubectl apply -f 02-deny-privileged-constraint.yaml
kubectl get constrainttemplates
kubectl get k8sdenyprivilegedcontainer
```
Test:
`bad-privileged-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-privileged-pod
  namespace: policy-demo
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        privileged: true
```

```sh
kubectl create namespace policy-demo
kubectl apply -f bad-privileged-pod.yaml
```

Expected result:
```text
Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request:
[deny-privileged-containers] Privileged container is not allowed: ubuntu
```

How this works:
```text
kubectl apply -f bad-privileged-pod.yaml
        |
        v
kube-apiserver
        |
        v
Gatekeeper admission webhook
        |
        v
OPA evaluates Rego policy
        |
        v
privileged == true
        |
        v
reject Pod
```

---

# Kubernetes Secrets

There are two primary methods to create a Kubernetes Secret: the imperative and declarative approaches.
**1. Imperative Approach**
```sh
kubectl create secret generic app-secret --from-literal=DB_Host=mysql --from-literal=DB_User=root --from-literal=DB_Password=paswrd
kubectl create secret generic app-secret --from-file=app_secret.properties
```

**2. Declarative Approach**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_Host: bXlzcWw=
  DB_User: cm9vdA==
  DB_Password: cGFzd3Jk
```

```sh
kubectl create -f secret-data.yaml
```

**Viewing and Decoding Secrets**
```sh
kubectl get secrets
kubectl get secret app-secret -o yaml
# To decode an encoded value, execute:
echo -n 'bXlzcWw=' | base64 --decode
```

### Injecting Secrets into a Pod
After creating a Secret, you can inject it into a Pod in two ways: as environment variables or as files via a mounted volume.
**1. Injecting as Environment Variables**
Use the `envFrom` property in your container specification to inject the Secret data as environment variables. For example:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
  labels:
    name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 8080
      envFrom:
        - secretRef:
            name: app-secret
```
All key-value pairs in the `app-secret` will be available as environment variables in your container.

**2. Injecting as Files in a Volume**
If you were to mount the secret as a volume in the pod, each attribute - each key in the secret is created as a file, with the value of the secret as its content:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  username: admin
  password: myStrongPassword123
```
Here we have two attributes:
```text
username = admin
password = myStrongPassword123
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-demo
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: secret-volume
          mountPath: "/etc/secrets"
          readOnly: true

  volumes:
    - name: secret-volume
      secret:
        secretName: app-secret
```
Kubernetes mounts the Secret at: `/etc/secrets`
Then it creates files like this:
```text
/etc/secrets/username
/etc/secrets/password
```

**Security Considerations**
When managing Secrets in Kubernetes, keep the following security best practices in mind:
- Kubernetes Secrets are **encoded but not encrypted**, meaning anyone with access can decode them using base64.
- Avoid checking in secret definition files to version control systems, such as GitHub.
- By default, Secrets stored in etcd are not encrypted. Consider **enabling encryption at rest for enhanced security**

### Enabling Encryption at Rest - [link](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- Encryption at rest in Kubernetes means encrypting Kubernetes API data while it is stored in etcd - kube-apiserver encrypts the data before writing it to etcd.
- Encryption happens in the `kube-apiserver` before writing to `etcd`, not inside `etcd` itself.

This example is for a kubeadm cluster, where the kube-apiserver runs as a **static pod under**: `/etc/kubernetes/manifests/kube-apiserver.yaml`
1. Create a Secret before encryption: 
```sh
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=superpassword

kubectl get secret db-secret -o yaml
```
**This is only base64, not real encryption.**

2. Check the Secret directly from etcd
```sh
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/db-secret | hexdump -C
```
Before enabling encryption, you may be able to see readable Secret values in the output (decoded format)

check if encryption at rest is enabled or not: 
```sh
ps aux | grep kube-api | grep "encryption-provider-config"
# or 
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep "encryption-provider-config"
```

3. Generate an encryption key
generate a 32-byte key and base64 encode it:
```sh
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
echo $ENCRYPTION_KEY
```

4. Create the EncryptionConfiguration file
Create this file on the control-plane node:
```sh
sudo mkdir -p /etc/kubernetes/enc
sudo tee /etc/kubernetes/enc/enc.yaml > /dev/null <<EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources: # 
  - resources:
	  # That means only the Secret objects are going to be encrypted , Not neccessarily encrypt the data about pods and deployments
      - secrets
    providers: # This is the list of encryption providers Kubernetes will use for the selected resource.
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY} # This is the actual encryption key. Here we can use <Base Encoded Secret> also.
      - identity: {} # it is used as a fallback for reading old unencrypted data.
EOF

# Set strict permissions:
sudo chmod 600 /etc/kubernetes/enc/enc.yaml
sudo chown root:root /etc/kubernetes/enc/enc.yaml
```

> [!note]
> - The first provider `aescbc` is used to encrypt new data, and all listed providers `identity` can be used to decrypt old data if needed.
> - If the `identity` is the first one , that means this is what is going to be used for encryption And since it's `identity`, **it's not going to encrypt anything at all**
> - So if we if you really want to encrypt the data in `etcd`, one of these `aescbc`, `aesgcm` should be at the top.

> [!note] 
> Now Kubernetes tries to read the Secret:
> 1. Try decrypt using `aescbc`
> 2. If that does not match, try `identity`
> 3. identity means read it as plain/unencrypted data

Important: `the identity: {}` provider means **no encryption.** We keep it at the bottom as a fallback during migration, so the API server can still read old unencrypted Secrets

> [!summary]
> 4. Try decrypt using aescbc
> 5. If that does not match, try identity
> 6. identity means read it as plain/unencrypted data

**Logic**:
```text
Read Secret from etcd
        |
        v
Is it encrypted with aescbc/key1?
        |
        | yes
        v
Decrypt using aescbc(key1)

Or:

Read Secret from etcd
        |
        v
Is it old unencrypted data?
        |
        | yes
        v
Read using identity provider
```

5. Edit the kube-apiserver manifest
```sh
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
# Under the kube-apiserver command list, add:
# - --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
```

And add a `volume` under `spec.volumes`:
```yaml
  volumes:
  - name: enc
    hostPath:
      path: /etc/kubernetes/enc # this is a Local Directory on the Control Plane node
      type: DirectoryOrCreate
```
Then add a `volumeMount` inside the `kube-apiserver` container:
```yaml
    volumeMounts:
    - mountPath: /etc/kubernetes/enc # this is a Container Directory
      name: enc
      readOnly: true
```

6. Wait for kube-apiserver to restart
Because kube-apiserver is a static pod, kubelet will restart it automatically after you save the manifest.
Check:
```sh
kubectl get pods -n kube-system | grep kube-apiserver
```

7. Create a new Secret after encryption
```sh
kubectl create secret generic encrypted-secret \
  --from-literal=password=mynewpassword
```
Now read it directly from etcd:
```sh
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/encrypted-secret | hexdump -C
```
You should see something like: `k8s:enc:aescbc:v1:key1` , That prefix means the Secret was encrypted with the `aescbc` provider and `key1`.

8. Encrypt old existing Secrets
Very important: enabling encryption only affects **new writes**. Existing Secrets already stored in etcd are not automatically rewritten. Kubernetes recommends replacing existing Secrets so the API server rewrites them encrypted
```sh
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```
This reads all Secrets and writes them back through the API server, so they get encrypted before being stored in etcd.

---

# Container Sandboxing
- Container sandboxing means running a container inside an extra isolation layer so it has less direct access to the host kernel.
- In normal Kubernetes, most containers run with a standard runtime like `runc`. That means the container is isolated using Linux features like: `namespaces`, `cgroups`, `capabilities`, `seccomp`, `AppArmor / SELinux`
- Normal containers still share the host kernel.
- So if a container has a dangerous kernel-level exploit, bad privileges, or weak configuration, it may be able to escape from the container and affect the node.
- Container sandboxing adds another security boundary between the container and the host.

**Normal container**:
```text
Container process
      |
      v
Host Linux kernel
      |
      v
Node hardware
```

**Sandboxed container**: 
```text
Container process
      |
      v
Sandbox layer
      |
      v
Host Linux kernel / Hypervisor
      |
      v
Node hardware
```

## gVisor
- gVisor is a container sandboxing technology from Google.
- gVisor creates an extra isolation layer between the container and the Linux kernel by intercepting system calls, thereby reducing the attack surface.
- In a normal container, the application talks directly to the host kernel through system calls
- With gVisor, many of those system calls go first to gVisor instead of directly to the host kernel
- gVisor can be slower than normal containers because system calls pass through an extra layer

**Main gVisor components**:
- **Sentry** 
	- acts like an application-level kernel for the container.
	- When the app inside the container makes a system call, for example: `open()`, `open()`, the call is handled by gVisor’s Sentry instead of going directly to the host kernel.

- **Gofer**
- Gofer handles filesystem access.
- The Sentry does not directly access normal host files. Instead, it talks to Gofer, and Gofer provides controlled access to filesystem resources.

> [!summary]
> - runc    = normal container runtime
> - gVisor  = sandboxed container runtime
> - runsc   = gVisor runtime binary
> - Sentry  = user-space application kernel
> - Gofer   = filesystem proxy

![gVisor](https://kodekloud.com/kk-media/image/upload/v1752871680/notes-assets/images/Certified-Kubernetes-Security-Specialist-CKS-gVisor/frame_200.jpg)

## Kata
- Kata Containers is a container sandboxing technology that runs containers inside lightweight virtual machines.
- Kata adds a stronger boundary: `Each Pod can run inside its own lightweight VM.`

## Container Runtime
When we run a Docker command, for example:
```sh
docker run nginx
```
- The command is first received by the **Docker CLI**. it converts the command into a **REST API request** and sends it to the **Docker daemon - dockerd**
- **Docker daemon** receives the request and checks whether the required image is already available on the local machine. If the image exists locally, Docker uses it directly. If the image does not exist, Docker pulls it from a **container registry**, such as Docker Hub, which is the **default registry**
- After the image is available, the **Docker daemon** asks **containerd** to start the container. **containerd** is responsible for managing the container lifecycle. It prepares the container image and converts it into an **OCI-compliant bundle**. OCI stands for Open Container Initiative, which defines standards for container images and runtimes.
- Then, **containerd** starts a **containerd-shim process**. The **shim** is responsible for keeping the container running **independently from the Docker daemon**. This means that even if the Docker daemon restarts, the container can continue running.
- After that, the **shim** calls the low-level container runtime called **runc**. **runc** is the default OCI runtime used by Docker. It is responsible for actually creating the container on the Linux host.
- Finally, **runc** communicates with the Linux kernel and creates the required isolation using: `namespaces`, `cgroups`, `capabilities`, `filesystem mounts`

```text
                         +----------------+
                         |   Docker CLI   |
                         +----------------+
                                  |
                                  v
                         +----------------+
                         |    REST API    |
                         +----------------+
                                  |
                                  v
+------------------------------------------------------+
|                    Docker Daemon                     |
|                                                      |
|   +------------+     +------------+     +----------+ |
|   |   Images   |     |  Volumes   |     | Networks | |
|   +------------+     +------------+     +----------+ |
+------------------------------------------------------+
          |
          | checks/pulls image
          |----------------------------------------------------+
          |                                                    |
          v                                                    v
+--------------------------------------------------+     +----------------------+
|                    containerd                    |     |       Registry       |
|                                                  |     |                      |
|              +-------------------+               |     |  +------+  +------+  |
|              | Manage Containers |               |     |  |Ubuntu|  |CentOS|  |
|              +-------------------+               |     |  +------+  +------+  |
+--------------------------------------------------+     |                      |
          |                                             |  +------+  +------+  |
          v                                             |  |Nginx |  |MySQL |  |
+--------------------------------------------------+     |  +------+  +------+  |
|                containerd-shim                   |     |                      |
+--------------------------------------------------+     |  +------+  +------+  |
          |                                             |  |HTTPD |  |Tomcat|  |
          v                                             |  +------+  +------+  |
+--------------------------------------------------+     +----------------------+
|                       runC                       |
|                                                  |
|              +-------------------+               |
|              |   Run Containers  |               |
|              +-------------------+               |
+--------------------------------------------------+
          |
          v
+-------------------+        +-------------------+
|    Namespaces     |        |      Cgroups      |
+-------------------+        +-------------------+
```

```sh
# run a container with kata sandbox
docker run --runtime kata -d nginx
# run a container with gVisor sandbox
docker run --runtime runsc -d nginx
```

## Runtime Class in Kubernetes
Assuming that we already have `gVisor` or `kata` installed on our Kubernetes nodes:
`gvisor.yaml`: 
```yaml
apiVersion: node.k8s.io/v1beta1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
```
or use `kata.yaml`:
```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata
```

```sh
kubectl create -f gvisor.yaml
# or 
kubectl create -f kata.yaml
```

Pod: 
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  runtimeClassName: gvisor
  # runtimeClassName: kata
  containers:
    - image: nginx
      name: nginx
```

To test that the NGINX container is isolated from the Linux kernel:
```sh
pgrep -a nginx
# If you run this command on the Kubernetes node/host, you may not see the nginx process when the Pod is running with gVisor.
# With normal runc, the container process is created directly on the host kernel.
# With gVisor, the container is running inside a sandbox managed by runsc. The nginx process is isolated inside the gVisor sandbox

# show gVisor processes:
pgrep -a runsc
```

---

# One way SSL vs Mutual SSL

## One-way SSL
When a client connects to a bank’s website using one-way SSL:
1. The client receives the bank’s public certificate.
2. The web browser verifies the certificate by ensuring the certificate authority (CA) that signed it is trusted. Browsers maintain a trust store with trusted CA public keys.
3. The browser then uses the bank’s public certificate to encrypt a symmetric key, which is sent securely to the bank.
4. The bank decrypts the symmetric key using its private key.

- This process uses asymmetric encryption initially to securely share a symmetric key. Once the symmetric key is established, both the client and the bank use it to encrypt and decrypt the data exchanged in their communication.
- In one-way SSL, **only the client verifies the server’s certificate**. **The bank does not authenticate the client certificate**; it instead relies on additional credentials (such as username, password, or client number) to confirm the user's identity.
- One-way SSL is widely used for internet-based services like email accounts, social media, and online banking where verifying the server's authenticity is the primary concern.

## Mutual TLS (mTLS)

Consider a scenario where no human user is entering credentials—imagine two organizations exchanging confidential information. For instance, suppose mybank.com (acting as a client) needs to retrieve data from the server abc-financials. In this case, it is essential for the server to verify that the request comes from the legitimate mybank.com. This is where mutual TLS comes into play.

When using mTLS, **both parties authenticate each other**. Here is how the process works when mybank.com (client) requests data from abc-financials (server):
1. The client requests the server’s public certificate.
2. The server responds with its public certificate and simultaneously requests the client’s certificate.
3. The client verifies the server’s certificate using the CA’s public keys from its trust store.
4. After successful verification, the client sends its certificate to the server along with a symmetric key encrypted with the server’s public key.
5. The server validates the client’s certificate using the CA to ensure it belongs to mybank.com

- Once both parties have mutually authenticated, they use the shared symmetric key to encrypt all subsequent communications, ensuring that data exchange remains secure.
- Mutual TLS is especially beneficial in scenarios where automated systems or organizations need to exchange data securely, as it provides strong authentication on both ends.

## Summary
The following table summarizes the key differences between one-way SSL and mutual TLS:

| Feature                  | One-way SSL                                                     | Mutual TLS                                                  |
| ------------------------ | --------------------------------------------------------------- | ----------------------------------------------------------- |
| Certificate verification | Only the server's certificate is verified by the client         | Both client and server certificates are verified            |
| Typical use cases        | Web services (online banking, email, social media)              | Automated system-to-system communication (B2B data sharing) |
| Security level           | Secure communication; user authentication is handled separately | Enhanced security with mutual authentication                |

---

# Overview of Multi-Tenancy in Kubernetes
- Multi-tenancy means multiple users, teams, applications, or customers share the same Kubernetes cluster, but each tenant should be isolated from the others.
- One Kubernetes cluster, Multiple teams/customers, Each team should only access its own resources
- Instead of creating a separate cluster for every team or application, companies may use one shared cluster to reduce cost and simplify management.
- Example:
	- Team A → namespace team-a
	- Team B → namespace team-b
	- Team C → namespace team-c
- Each team can deploy its own workloads, but they should not be able to affect or access the others.

## Important Kubernetes controls
1. Namespaces
2. RBAC
3. ResourceQuota and LimitRange: These prevent one tenant from using all cluster resources.
	Example:
	```yaml
	apiVersion: v1
	kind: ResourceQuota
	metadata:
	  name: tenant-a-quota
	  namespace: tenant-a
	spec:
	  hard:
	    requests.cpu: "2"
	    requests.memory: 4Gi
	    limits.cpu: "4"
	    limits.memory: 8Gi
	    pods: "10"
	``` 

4. NetworkPolicy: By default, Pods in Kubernetes can communicate with each other unless you restrict them.
5. Pod Security: Use Pod Security Admission or policies to prevent dangerous workloads

## Soft vs hard multi-tenancy
**Soft multi-tenancy**
Used when tenants are internal teams and mostly trusted.
Example:
```text
One company
Multiple teams
Shared cluster
Namespaces + RBAC + quotas + network policies
```

**Hard multi-tenancy**
Used when tenants are untrusted or external customers.
Example:
```text
SaaS platform
Multiple customers
Strong isolation required
```
For hard multi-tenancy, namespace isolation may not be enough. You may need:
```text
Separate clusters
Virtual clusters
Dedicated node pools
Sandboxed runtimes like gVisor/Kata
Stronger network and policy controls
```

## Kubernetes multi-tenancy has two main models
1. **Multi-team tenancy**
	- This is when multiple internal teams inside the same company share one Kubernetes cluster.
	- Company Kubernetes Cluster
		├── frontend-team namespace
		├── backend-team namespace
		├── data-team namespace
		└── security-team namespace
	- In this model, the tenants are usually trusted internal teams.
	- Multi-team tenancy usually needs soft isolation.
	- That means you should still use:
		- Namespaces
		- RBAC
		- ResourceQuota
		- LimitRange
		- NetworkPolicy
		- Pod Security Admission
	
2. **Multi-customer tenancy**
	- This is when one Kubernetes cluster runs workloads for multiple external customers.
	- Example: 
		- SaaS Platform Kubernetes Cluster
			├── customer-a app
			├── customer-b app
			├── customer-c app
			└── customer-d app
	- The customers usually do not access Kubernetes directly.
	- They only use the application, dashboard, or API.
	- Multi-customer tenancy needs stronger isolation because customers are external and less trusted.
	- You may need: 
		- Strict namespace isolation
		- NetworkPolicy deny-by-default
		- Dedicated node pools
		- Taints and tolerations
		- Separate service accounts
		- Separate databases or schemas
		- Encrypted secrets
		- Strong admission policies
		- Runtime sandboxing like gVisor or Kata
		- Separate clusters for high-risk customers

**Remember this simple idea**: 
- **Multi-team tenancy** = internal teams sharing one cluster
- **Multi-customer tenancy** = external customers sharing app infrastructure

## Levels of Isolation in Kubernetes: Namespace, Pod, Node
Kubernetes isolation can happen at different layers. The stronger the isolation layer, the better the security, but usually with more cost and complexity.
- Weakest isolation  → Namespace
- Medium isolation   → Pod / workload isolation
- Stronger isolation → Node isolation
- Strongest option   → Separate cluster

1. **Namespace isolation**
	- Namespaces help organize resources, but they do not automatically stop:
		- Pods from talking to other namespaces
		- Users with broad RBAC from accessing other namespaces 
		- One team consuming too many cluster resources
		- Privileged containers from escaping security boundaries
	- So with namespaces, you should also use:
		- RBAC
		- ResourceQuota
		- LimitRange
		- NetworkPolicy
		- Pod Security Admission
	- Example: 
		- kubectl create namespace tenant-a
		- kubectl create namespace tenant-b
		- Then restrict access with RBAC: User A → can access only tenant-a , User B → can access only tenant-b
2. **Pod isolation**
	- Do not run privileged containers
	- Do not allow hostPath unless required
	- Do not use hostNetwork unless required
	- Run containers as non-root
	- Drop unnecessary Linux capabilities
	- Use readOnlyRootFilesystem
	- Use seccomp / AppArmor / SELinux
	
		```yaml
		apiVersion: v1
		kind: Pod
		metadata:
		  name: secure-pod
		spec:
		  securityContext:
		    runAsNonRoot: true
		    seccompProfile:
		      type: RuntimeDefault
		  containers:
		    - name: app
		      image: nginx
		      securityContext:
		        allowPrivilegeEscalation: false
		        readOnlyRootFilesystem: true
		        capabilities:
		          drop:
		            - ALL
		```

3. **Node isolation**
	- Node isolation means dedicating specific worker nodes for specific tenants, teams, or workload types.
	- Example: 
		- node-pool-a → tenant-a workloads only
		- node-pool-b → tenant-b workloads only
		- node-pool-prod → production workloads only
		- node-pool-dev → development workloads only
	- You can implement this using:
		- Node labels
		- nodeSelector
		- Node affinity
		- Taints and tolerations
		- Admission policies

		```sh
		kubectl label node worker1 tenant=team-a
		kubectl taint node worker1 tenant=team-a:NoSchedule
		```


		```yaml
		apiVersion: v1
		kind: Pod
		metadata:
		  name: team-a-app
		spec:
		  nodeSelector:
		    tenant: team-a
		  tolerations:
		    - key: "tenant"
		      operator: "Equal"
		      value: "team-a"
		      effect: "NoSchedule"
		  containers:
		    - name: app
		      image: nginx
		```

## Control Plane Isolation​
- Control plane isolation = isolate Kubernetes API access and API resources
- Main tools:
	- Namespaces
	- RBAC
	- ResourceQuota
	- LimitRange

**Example: Resource Quota for a Namespace**
The following example demonstrates a Kubernetes resource quota configuration that restricts the resource consumption for a specific namespace named `namespace_a`. This setup limits resource requests to 4 CPUs and 16 GiB of memory, while capping resource limits at 8 CPUs and 32 GiB of memory:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: cpu-memory-quota
  namespace: namespace_a
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 16Gi
    limits.cpu: "8"
    limits.memory: 32Gi
```

**Example: Pod with Resource Restrictions**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-limited-pod
  namespace: namespace_a
spec:
  containers:
  - name: resource-limited-container
    image: nginx
    resources:
      requests:
        cpu: "250m"
        memory: "64Mi"
      limits:
        cpu: "500m"
        memory: "128Mi"
```

## Data Plane Isolation​
Data-plane isolation is implemented through several critical mechanisms, including:
- **Network Policies**: These policies control the flow of traffic between pods, ensuring that only authorized communications occur.
- **Storage Isolation**: Storage resources are segmented to prevent unauthorized access and to guarantee that storage operations of one tenant do not interfere with others.
- **Taints and Tolerations**: This mechanism prevents pods from being scheduled on inappropriate nodes by allowing only those pods with the matching tolerations to run on tainted nodes.

### Data Plane Isolation Network
Below is an example YAML manifest for a network policy that permits pods labeled with `app: backend` in the `tenant_a` namespace to receive `TCP` traffic on port `8080` from any pod in a namespace labeled as `tenant_b`:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-tenant-a-to-tenant-b
  namespace: tenant_a
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tenant: tenant_b
  ports:
  - protocol: TCP
    port: 8080
```

### Data Plane Isolation Storage
Consider the following scenario with two namespaces:
- **Namespace A**: Dedicated to a critical tenant requiring high-performance storage.
- **Namespace B**: Allocated to a regular tenant with standard resource demands.

For the critical tenant in `Namespace A`:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-performance
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1                          # AWS io1 disks support high IOPS
  iopsPerGB: "50"                    # Specify high IOPS per GB
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

For the regular tenant in `Namespace B`: 
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-performance
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2        # AWS gp2 disks support general-purpose workloads
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

### Data Plane Node Pools and TaintsTolerations for Isolation
- Node isolation using taints and tolerations prevents unauthorized pods from being scheduled on nodes that are reserved for a particular tenant.

**Step 1: Apply a Taint to the Node**
```sh
kubectl taint nodes nodeA customer=customerA:NoSchedule
```

**Step 2: Configure Pod Definitions with Tolerations**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: customer-a-pod
  namespace: customer_a
spec:
  containers:
  - name: customer-a-container
    image: nginx
  tolerations:
  - key: "customer"
    operator: "Equal"
    value: "customerA"
    effect: "NoSchedule"
```

## Additional Considerations​ - API Priority & Fairness
- In multi-tenancy, one tenant/team may create too many API requests and affect other tenants. API Priority and Fairness - APF, helps prevent one user, controller, or workload from starving others by controlling how API requests are classified, queued, and executed: 
- **The problem APF solves**: Without protection, one tenant can overload the API server.
	```yaml
		Tenant A sends many API requests
		        ↓
		kube-apiserver becomes busy
		        ↓
		Tenant B kubectl/app/controller becomes slow
		        ↓
		Cluster control plane stability is affected
	```

- APF classifies incoming requests and assigns them to different priority levels.
	```text
	Request comes to kube-apiserver
	        ↓
	FlowSchema decides which group it belongs to
	        ↓
	PriorityLevelConfiguration decides how much API capacity it gets
	        ↓
	Request is executed, queued, or rejected
	```

- Kubernetes uses **FlowSchemas** to classify requests and **PriorityLevelConfigurations** to define how much concurrency each priority level receives

### Configuring API Priority
- This config is for Kubernetes API Priority and Fairness APF. It controls how much kube-apiserver capacity different request groups get.
- This does not control Pod CPU/memory, and it does not control Pod scheduling priority. It controls requests to the Kubernetes API server
- `PriorityLevelConfiguration` = defines API server capacity/queue behavior - It controls how much kube-apiserver concurrency a priority level gets, and what happens when the limit is reached: queue or reject - **API server traffic priority profile**
	- high-priority → gets more API server concurrency
	- low-priority  → gets less API server concurrency
- `FlowSchema` = decides which requests go to which priority level - It decides which users, groups, service accounts, verbs, resources, or namespaces should use a specific priority level.

1. **PriorityLevelConfiguration**
	```yaml
	apiVersion: flowcontrol.apiserver.k8s.io/v1
	kind: PriorityLevelConfiguration
	metadata:
	  name: high-priority
	spec:
	  type: Limited
	  limited:
	    nominalConcurrencyShares: 10 # concurrency means the number of requests the API server is actively handling at the same moment — requests that are "in flight" — not requests per second. High priority gets more concurrency
	    limitResponse: # what happens to requests that can't be served immediately:
	      type: Queue
	      queuing:
	        queues: 64  # Create 64 waiting queues for this priority level.
	        handSize: 8 # For each request flow, Kubernetes picks 8 queues from the 64.
	        queueLengthLimit: 50 # Each queue can hold maximum 50 waiting requests.
	---
	apiVersion: flowcontrol.apiserver.k8s.io/v1
	kind: PriorityLevelConfiguration
	metadata:
	  name: low-priority
	spec:
	  type: Limited
	  limited:
	    nominalConcurrencyShares: 1 # Low priority gets less concurrency
	    limitResponse:
	      type: Queue
	      queuing:
	        queues: 64
	        handSize: 8
	        queueLengthLimit: 50
	```
	So high-priority gets around `10` times more API server request capacity than low-priority, depending on the other `PriorityLevelConfigurations` in the cluster.


2. **FlowSchema**
	`FlowSchema` maps incoming kube-apiserver requests to a `PriorityLevelConfiguration`
	```yaml
	apiVersion: flowcontrol.apiserver.k8s.io/v1
	kind: FlowSchema
	metadata:
	  name: high-priority-namespace-a
	spec:
	  matchingPrecedence: 1000 # This controls the order of FlowSchema matching. Lower number = checked earlier (1000 is checked before 2000)
	  priorityLevelConfiguration:
	    name: high-priority
	  distinguisherMethod:
	    type: ByNamespace # This means Kubernetes separates traffic by namespace for fairness.
	  rules:
	    - subjects: # Who is sending the API request?
	        - kind: ServiceAccount
	          serviceAccount:
	            name: system-account
	            namespace: namespace-a # Match requests coming from this ServiceAccount: namespace-a/system-account
	      resourceRules: # What is the request trying to access: 
	        - verbs: ["*"] # any action: get, list, watch, create, update, delete
	          apiGroups: ["*"] # any API group
	          resources: ["*"] # any resource: pods, deployments, secrets, services, configmaps, etc.
	          namespaces: ["namespace-a"] # only resources inside namespace-a
	---
	apiVersion: flowcontrol.apiserver.k8s.io/v1
	kind: FlowSchema
	metadata:
	  name: low-priority-namespace-b
	spec:
	  matchingPrecedence: 2000
	  priorityLevelConfiguration:
	    name: low-priority
	  distinguisherMethod:
	    type: ByNamespace
	  rules:
	    - subjects: # Who is sending the API request?
	        - kind: Group
	          group:
	            name: regular-users # This rule matches requests coming from Group: regular-users
	        - kind: ServiceAccount
	          serviceAccount:
	            name: default
	            namespace: namespace-b # this rule matches requests coming from this ServiceAccount: namespace-b/default ServiceAccount
	      resourceRules:
	        - verbs: ["*"]
	          apiGroups: ["*"]
	          resources: ["*"]
	          namespaces: ["namespace-b"]
	```
	Simple meaning:
	```text
		If the request comes from:

		ServiceAccount: system-account
		Namespace: namespace-a
		
		And the request is targeting resources inside:
		
		namespace-a
		
		Then send this request to:
		
		high-priority
	```

### Pod Priority and Preemption
- Pod Priority tells Kubernetes: **This Pod is more important than that Pod.**
- If the cluster does not have enough CPU/memory for a high-priority Pod, Kubernetes may remove lower-priority Pods to make space.
	```text
		High-priority Pod is pending
		        ↓
		No node has enough free resources
		        ↓
		Scheduler checks lower-priority Pods
		        ↓
		It may preempt/delete lower-priority Pods
		        ↓
		High-priority Pod gets scheduled
	```

1. **PriorityClass config**
	```yaml
	apiVersion: scheduling.k8s.io/v1
	kind: PriorityClass
	metadata:
	  name: high-priority
	value: 1000 # Higher number = more important.
	globalDefault: false # Do not automatically give this priority to all Pods. A Pod must explicitly use it: priorityClassName: high-priority .. If globalDefault: true, Pods without priorityClassName would get that class by default.
	description: "This priority class is for critical production workloads."
	preemptionPolicy: Never # This Pod has high scheduling priority, but it will NOT kill/remove lower-priority Pods to make space.
	---
	apiVersion: scheduling.k8s.io/v1
	kind: PriorityClass
	metadata:
	  name: low-priority
	value: 100
	globalDefault: false
	description: "This priority class is for non-critical development workloads."
	```

2. **Pod using high priority**
	```yaml
	apiVersion: v1
	kind: Pod
	metadata:
	  name: critical-app
	  namespace: namespace-a
	spec:
	  priorityClassName: high-priority
	  containers:
	    - name: app-container
	      image: nginx
	      resources:
	        requests:
	          memory: "500Mi"
	          cpu: "500m"
	        limits:
	          memory: "500Mi"
	          cpu: "500m"
	```
	- This Pod is critical.
	- Schedule it before lower-priority Pods.
	- If needed, preempt lower-priority Pods.
	
	Pod Priority does not magically create resources, It only helps Kubernetes choose:
	- Which Pod should run first?
	- Which lower-priority Pod can be removed if needed?

### Comparing API Priority and Pod Priority
- **API Priority and Fairness**:
	- These settings control the flow and processing of Kubernetes API requests. They manage operations such as creating, updating, or fetching cluster resources and ensure that critical API interactions are not stalled by heavy traffic from lower priority sources.
	
- **Pod Priority and Preemption**:
	- These mechanisms focus on resource allocation at the node level. They prioritize scheduling for critical pods and allow the system to evict lower priority pods when essential resources are required.

| Feature  | API Priority and Fairness                                             | Pod Priority and Preemption                     |
| -------- | --------------------------------------------------------------------- | ----------------------------------------------- |
| Scope    | API server request handling                                           | Pod scheduling and resource allocation on nodes |
| Purpose  | Ensures fair processing of API requests                               | Ensures critical pods are scheduled and run     |
| Controls | API traffic like `kubectl get`, `kubectl create`, controller requests | Resource allocation (CPU/memory) and eviction   |
| Handles  | Fair access to the Kubernetes API server                              | Fair usage of node resources                    |

---

# Quality of Service
- Quality of Service = how Kubernetes classifies Pods based on CPU/memory requests and limits
- How Kubernetes implements Quality-of-Service to manage resource allocation and ensure fair distribution among tenants in multi-tenant environments: 

## Different QoS classes in Kubernetes:
### Guaranteed QoS
- Pods receive the Guaranteed QoS classification when both CPU and memory requests and limits are set to identical values. 
- With this configuration, Kubernetes guarantees that the pod always has the specified resources. 
- The pod will not be evicted unless the host node experiences instability.

For example, consider a pod in namespace A running **mission-critical production workloads**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
  namespace: namespace-a
spec:
  containers:
    - name: critical-container
      image: nginx
      resources:
        requests:
          memory: "500Mi"
          cpu: "500m"
        limits:
          memory: "500Mi"
          cpu: "500m"
```
- In this configuration, both the requests and limits for memory and CPU are identical (500Mi and 500m, respectively), ensuring that the pod has a fixed allocation of resources.
- This Pod is the most protected from eviction. 

### Burstable QoS
- Pods that set resource requests to values lower than their limits belong to the Burstable QoS class. 
- This configuration guarantees a minimum level of resources while allowing the pod to burst beyond that level if additional resources are available. 
- This is particularly useful for workloads that can tolerate variability in resource consumption.

For instance, consider a pod in namespace B running **burstable, less critical workloads**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable-app
  namespace: namespace-b
spec:
  containers:
    - name: burstable-container
      image: nginx
      resources:
        requests:
          memory: "200Mi"
          cpu: "200m"
        limits:
          memory: "16Gi"
          cpu: "1"
```
Here, the pod is guaranteed a minimum of 200Mi memory and 200m CPU but can scale up to 16Gi memory and 1 CPU if additional resources are available.

### Best-Effort QoS
- Pods that do not specify any resource requests or limits fall under the Best-Effort QoS class. 
- These pods do not benefit from guaranteed resources and will only consume resources when they are available. 
- Under high resource pressure, Best-Effort pods are the first candidates to be evicted. 
- This QoS class is generally appropriate for **development and testing workloads where resource guarantees are less critical**.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-app
  namespace: namespace-c
spec:
  containers:
    - name: test-container
      image: nginx
```
- It is usually the first to be evicted when the node has resource pressure.

**QoS helps with that decision**:
- BestEffort  → evicted first
- Burstable   → evicted after BestEffort
- Guaranteed  → evicted last

**How to check Pod QoS**: 
```sh
kubectl get pod <pod-name> -o yaml
# Then look for:
# status:
#   qosClass: Guaranteed
```

> [!important]
> 
> **QoS**
> - based on requests and limits
> - affects eviction order during node pressure
> 
> **Pod Priority**
> - based on PriorityClass
> - affects scheduling and preemption
> 
> **QoS decides**: which Pod is safer to evict under pressure?
> **Pod Priority decides**: which Pod is more important to schedule?

- Can QoS cancel eviction? No.
- Can QoS reduce eviction chance? Yes.
- Best QoS for protection? Guaranteed.
- Best full protection? High PriorityClass + Guaranteed QoS + enough node resources.

[understanding-kubernetes-cpu-limits-and-requests](https://eqraatech.com/understanding-kubernetes-cpu-limits-and-requests/)

### Network QoS
Although Kubernetes does not provide network QoS directly, it can be implemented using Container Network Interface (CNI) plugins such as Calico, or by leveraging Linux traffic control. Network QoS manages the network bandwidth usage of pods, which is essential when tenants have differing network performance requirements. For example, a network policy implemented with Calico for namespace A might look like this:
```yaml
apiVersion: crd.projectcalico.org/v1
kind: NetworkPolicy
metadata:
  name: tenant-a-network-policy
  namespace: namespace-a
spec:
  selector: all()  # Apply to all pods in Namespace A
  ingress:
    - action: Allow
      protocol: TCP
      destination:
        ports: [80]
      limits:
        rate: 10Mbps  # Limit ingress traffic to 10Mbps for tenant pods
  egress:
    - action: Allow
      protocol: TCP
      destination:
        ports: [80]
      limits:
        rate: 10Mbps  # Limit egress traffic to 10Mbps for tenant pods
```

### Storage QoS
Storage QoS governs the number of I/O operations a pod can perform on its volumes. This is critical for workloads like databases that require high disk performance. Kubernetes indirectly supports storage QoS using storage classes and the underlying QoS features provided by your storage platform. For a high-performance database workload in namespace A, you might use a StorageClass configured for high IOPS:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-performance
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1            # AWS io1 disks support high IOPS
  iopsPerGB: "50"      # Specify high IOPS per GB
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
```
In contrast, namespace B, which may run development workloads, could use a standard storage class with lower IOPS that is adequate for general-purpose applications:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-performance
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2            # AWS gp2 disks support general-purpose workloads 
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

---

# DNS in Multi-Tenant Environments
- Kubernetes uses DNS so Pods can find Services by name instead of IP address. For example, a Pod can access a Service using a name like: `my-service.my-namespace.svc.cluster.local`
- Kubernetes creates DNS records for Services and Pods, and workloads use DNS to discover Services inside the cluster

**The problem in multi-tenancy**
- By default, Kubernetes DNS can resolve Services across namespaces.
- Example: `tenant-a Pod → can resolve service in tenant-b namespace` , Example DNS query: `curl http://api-service.tenant-b.svc.cluster.local`

## CoreDNS role
- Most Kubernetes clusters use CoreDNS for internal DNS.
- Default behavior can expose records from all namespaces unless restricted. The CoreDNS Kubernetes plugin has a `namespaces` option that only exposes selected namespaces; if omitted, all namespaces are exposed.
- With `namespaces`:
	- Only expose DNS records for `tenant-a` namespace to `tenant-a` DNS scope.
	- Only expose DNS records for `tenant-b` namespace to `tenant-b` DNS scope.

**Configuring CoreDNS for Tenant Isolation**: 
Use the following command to edit the CoreDNS configuration stored in the ConfigMap:
```sh
kubectl edit configmap coredns -n kube-system
```
adding the `fallthrough in-namespace`: 
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods verified
            fallthrough in-namespace # Only resolve if the service or pod exists in the same namespaces
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```
After updating the ConfigMap, CoreDNS automatically reloads the configuration in its running pods.

**Test the DNS Restrictions**: 
Deploy pods in different namespaces and perform DNS queries to verify that cross-namespace resolution is restricted. For instance, use the command below to launch a test pod in `namespace-a` and attempt to resolve a service in `namespace-b`:
```sh
kubectl run test-pod --rm -i --tty --image=busybox --restart=Never --namespace=namespace-a -- nslookup backend.namespace-b.svc.cluster.local
```


---

# Pod-to-Pod Encryption
- By default, Apps communicate like this: `Pod A  ---- plain HTTP/TCP ---->  Pod B`
- With Pod-to-Pod encryption: `Pod A  ---- encrypted traffic ---->  Pod B`
- Normal TLS: Client verifies server
- With mTLS
	- Client verifies server
	- Server verifies client
	- Traffic is encrypted
- So both Pods prove their identity before communicating.

## Implement pod to pod encryption by use of mTLS:
- Encrypting communication between Pods using mTLS, usually through a **service mesh** like `Istio` or `Linkerd`. The main idea is that your application does not need to implement TLS itself. Instead, the service mesh adds encryption around the Pod traffic.

### How mTLS Works Between Pods
The process of establishing a secure connection between two pods involves several steps:
1. **Certificate Request:**
   Pod A initiates communication by requesting Pod B’s certificate.
2. **Certificate Exchange:**
   Pod B responds with its certificate and simultaneously requests Pod A’s certificate.
3. **Certificate Validation and Key Exchange:**
   After validating Pod B's certificate, Pod A sends its public certificate along with a symmetric encryption key.
4. **Secure Communication:**
   Once Pod B validates Pod A’s certificate, both pods switch to using the symmetric key to encrypt all further communications.

## Demo: Authentication
```sh
root@controlplane ~ ➜  kubectl label ns default istio-injection=enabled
# Any new Pod created in namespace default should automatically get an Istio sidecar container.
# So when you create a Pod in test, Istio injects an extra container called:
# istio-proxy

root@controlplane ~ ➜  kubectl get ns --show-labels
NAME              STATUS   AGE   LABELS
default           Active   80m   istio-injection=enabled,kubernetes.io/metadata.name=default
istio-system      Active   62s   kubernetes.io/metadata.name=istio-system
kube-node-lease   Active   80m   kubernetes.io/metadata.name=kube-node-lease
kube-public       Active   80m   kubernetes.io/metadata.name=kube-public
kube-system       Active   80m   kubernetes.io/metadata.name=kube-system

root@controlplane ~ ➜  k apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/helloworld/helloworld.yaml
service/helloworld created
deployment.apps/helloworld-v1 created
deployment.apps/helloworld-v2 created

root@controlplane ~ ➜  kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
helloworld-v1-5dd8856698-6l2zk   2/2     Running   0          15s
helloworld-v2-56df4c568b-qzzww   2/2     Running   0          15s

root@controlplane ~ ➜  k get svc 
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
helloworld   ClusterIP   10.96.165.111   <none>        5000/TCP   18s
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP    81m

root@controlplane ~ ➜  k create ns test 
namespace/test created

root@controlplane ~ ➜  k run test --image=nginx -n test 
pod/test created

root@controlplane ~ ➜  kubectl exec -ti -n test test -- curl helloworld.default.svc.cluster.local:5000/hello
Hello version: v1, instance: helloworld-v1-5dd8856698-6l2zk

root@controlplane ~ ➜  kubectl exec -ti -n test test -- curl --head helloworld.default.svc.cluster.local:5000/hello
HTTP/1.1 200 OK
server: istio-envoy
date: Fri, 19 Jun 2026 17:46:31 GMT
content-type: text/html; charset=utf-8
content-length: 60
x-envoy-upstream-service-time: 85
x-envoy-decorator-operation: helloworld.default.svc.cluster.local:5000/*

# Create and apply a global PeerAuthentication policy to enforce STRICT mTLS mode
root@controlplane ~ ✖ cat peer_auth_global.yaml 
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
	
# mode: STRICT
# All workloads in the mesh (such as: default namespace) must receive traffic using Istio mTLS.
# Plain HTTP/plain TCP from non-mesh Pods will be rejected.

# mode: Permissive
# Istio accepts BOTH:
# 1. encrypted Istio mTLS traffic
# 2. plain text traffic from non-Istio Pods
   
root@controlplane ~ ➜  kubectl apply -f peer_auth_global.yaml
peerauthentication.security.istio.io/default created

root@controlplane ~ ➜  k get peerauthentications.security.istio.io 
No resources found in default namespace.

root@controlplane ~ ➜  k get peerauthentications.security.istio.io -n istio-system 
NAME      MODE     AGE
default   STRICT   20s

root@controlplane ~ ➜
root@controlplane ~ ➜  kubectl exec -ti -n test test -- curl --head http://helloworld.default.svc:5000/hello
curl: (56) Recv failure: Connection reset by peer
command terminated with exit code 56

# How do we fix this?
root@controlplane ~ ✖ kubectl delete pod -n test test
pod "test" deleted

root@controlplane ~ ➜  kubectl label ns test istio-injection=enabled
namespace/test labeled

root@controlplane ~ ➜  k get ns --show-labels 
NAME              STATUS   AGE     LABELS
default           Active   86m     istio-injection=enabled,kubernetes.io/metadata.name=default
istio-system      Active   6m55s   kubernetes.io/metadata.name=istio-system
kube-node-lease   Active   86m     kubernetes.io/metadata.name=kube-node-lease
kube-public       Active   86m     kubernetes.io/metadata.name=kube-public
kube-system       Active   86m     kubernetes.io/metadata.name=kube-system
test              Active   3m48s   istio-injection=enabled,kubernetes.io/metadata.name=test

root@controlplane ~ ➜  kubectl run test --image=nginx -n test
pod/test created

root@controlplane ~ ➜  kubectl exec -ti -n test test -- curl http://helloworld.default.svc:5000/hello
Hello version: v1, instance: helloworld-v1-5dd8856698-6l2zk

root@controlplane ~ ➜  kubectl exec -ti -n test test -- curl --head http://helloworld.default.svc:5000/hello
HTTP/1.1 200 OK
server: envoy
date: Fri, 19 Jun 2026 17:49:37 GMT
content-type: text/html; charset=utf-8
content-length: 60
x-envoy-upstream-service-time: 83
```

---

# Cilium
Cilium is an open-source, cloud-native solution for:
- Networking
- Security
- Observability

between workloads such as Kubernetes Pods, containers, and services. Cilium’s own site describes it as a cloud-native solution for providing, securing, and observing network connectivity between workloads

**Cilium is built on eBPF**: eBPF allows programs to run safely inside the Linux kernel. This lets Cilium inspect, route, filter, and secure traffic efficiently without depending only on traditional iptables rules.
```text
Traditional CNI:
  Often uses iptables heavily

Cilium:
  Uses eBPF inside the Linux kernel
```
Cilium stands out by leveraging eBPF, a cutting-edge technology that runs directly within the Linux kernel. This integration enables efficient networking with advanced security measures. At its core, Cilium manages pod networking using the Container Network Interface (CNI), ensuring seamless communication among pods while enforcing strict security policies to allow only authorized traffic.

**Key Features of Cilium**
Cilium offers several robust features to optimize your Kubernetes environment:
- **Network Policies**: Secure pod-to-pod communications by enforcing policies that prevent unauthorized access.
- **Services and Load Balancing**: Efficiently route traffic and distribute loads evenly among pods, ensuring high availability.
- **Bandwidth Management**: Regulate traffic consumption per pod to prevent network congestion.
- **Flow and Policy Logging**: Monitor network traffic in real time and gain insights into policy enforcement across your cluster.
- **Security and Operational Metrics**: Obtain deep visibility into your cluster’s security posture and overall performance.

![Cilium Architecture](https://kodekloud.com/kk-media/image/upload/v1752871675/notes-assets/images/Certified-Kubernetes-Security-Specialist-CKS-Understanding-Ciliums-Architecture/frame_60.jpg)

## Cilium network policy
Below is the YAML configuration for the Cilium network policy, It controls which Pods can send traffic to which Pods:
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-traffic
spec:
  endpointSelector:
    matchLabels:
      app: myapp
  egress:
  - toEndpoints:
    - matchLabels:
        app: myapp
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
```

## Installing Cilium and Enabling Encryption
```sh
# Add the Cilium Helm repository
helm repo add cilium https://helm.cilium.io/

# Install Cilium with encryption enabled
helm install cilium cilium/cilium --version v1.18.0-pre.0 \
  --namespace kube-system \
  --set encryption.enabled=true \
  --set encryption.type=wireguard #  Wireguard: A VPN protocol that provides encrypted network communication
  
root@controlplane ~ ➜  k get po -n kube-system | grep cilium 
cilium-7lqdj                           1/1     Running   0              41s
cilium-envoy-74vgq                     1/1     Running   0              41s
cilium-envoy-hbg7j                     1/1     Running   0              41s
cilium-gh7dk                           0/1     Running   0              41s
cilium-operator-76b84cb496-cx2dp       1/1     Running   0              41s
cilium-operator-76b84cb496-lcntm       1/1     Running   0              41s

# Wait for the status of cilium to be OK
root@controlplane ~ ➜  cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium                   Desired: 2, Ready: 2/2, Available: 2/2
DaemonSet              cilium-envoy             Desired: 2, Ready: 2/2, Available: 2/2
Deployment             cilium-operator          Desired: 2, Ready: 2/2, Available: 2/2
Containers:            cilium                   Running: 2
                       cilium-envoy             Running: 2
                       cilium-operator          Running: 2
                       clustermesh-apiserver    
                       hubble-relay             
Cluster Pods:          2/2 managed by Cilium
Helm chart version:    1.18.0-pre.0
Image versions         cilium             quay.io/cilium/cilium:v1.18.0-pre.0@sha256:88711d5016c6969e47e92e5f499ccd80e3df93ab52bdd7bc321c2b4a6a434a9e: 2
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.32.3-1740976227-c3c35d52ca3b699de1f9448ab7174a9bdcb13f69@sha256:92af281c1904a1b7afab350a7bb0c6eee6cb2383eb74734b7cdd181927b8c034: 2
                       cilium-operator    quay.io/cilium/operator-generic:v1.18.0-pre.0@sha256:b1f1d0b3278efd8f9774d87d317f25a145fd165f22b19403bc6b92db750cea8a: 2

# Check the encryption status of the Cilium installation
root@controlplane ~ ➜  cilium encryption status
Encryption: Wireguard (2/2 nodes)
```

```sh
root@controlplane ~ ➜  cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
EOF

deployment.apps/nginx created
service/nginx created
```

```sh
root@controlplane ~ ➜  ls 
snap

root@controlplane ~ ➜  k get po 
NAME                     READY   STATUS    RESTARTS   AGE
nginx-56c45fd5ff-9p646   1/1     Running   0          24s

root@controlplane ~ ➜  k get svc 
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   3h3m
nginx        ClusterIP   10.103.72.32   <none>        80/TCP    26s

root@controlplane ~ ➜  kubectl run curlpod --image=rapidfort/curl --command -- sleep 3600
pod/curlpod created

root@controlplane ~ ➜  kubectl label pod curlpod app=curlpod
pod/curlpod labeled

$ watch kubectl exec -it curlpod -- curl -s http://nginx
return the HTML content of the NGINX welcome page

# Check that WireGuard has been enabled (number of peers should correspond to a number of nodes subtracted by one):
$ kubectl -n kube-system exec -ti ds/cilium -- bash
root@node01:/home/cilium# cilium-dbg status | grep Encryption
Encryption:              Wireguard       [NodeEncryption: Disabled, cilium_wg0 (Pubkey: T/v4JHQgwT8uIyAMyMThlufdn6kFDRMD5j4YRAVmFVE=, Port: 51871, Peers: 1)]

# Check that traffic is sent via the cilium_wg0 tunnel device is encrypted:
root@node01:/home/cilium# apt-get update && apt-get -y install tcpdump
root@node01:/home/cilium# tcpdump -n -i cilium_wg0 -X
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on cilium_wg0, link-type RAW (Raw IP), snapshot length 262144 bytes
19:35:49.612931 IP 192.168.121.87.33954 > 192.168.121.245.8472: OTV, flags [I] (0x08), overlay 0, instance 6
IP 10.0.0.34.36484 > 10.0.1.231.4240: Flags [F.], seq 3769716900, ack 1894771834, win 508, options [nop,nop,TS val 3172660624 ecr 3783114269], length 0
        0x0000:  4500 0066 ca10 0000 4011 3bd9 c0a8 7957  E..f....@.;...yW
        0x0010:  c0a8 79f5 84a2 2118 0052 0000 0800 0000  ..y...!..R......
        0x0020:  0000 0600 5a10 a0aa 2614 5a10 a0aa 2614  ....Z...&.Z...&.
        0x0030:  0800 4500 0034 8bf3 4000 4006 98c8 0a00  ..E..4..@.@.....
        0x0040:  0022 0a00 01e7 8e84 1090 e0b1 50a4 70ef  ."..........P.p.
        0x0050:  ec7a 8011 01fc 162f 0000 0101 080a bd1a  .z...../........
        0x0060:  f590 e17d be1d                           ...}..
19:35:49.613773 IP 192.168.121.245.38395 > 192.168.121.87.8472: OTV, flags [I] (0x08), overlay 0, instance 4
IP 10.0.1.231.4240 > 10.0.0.34.36484: Flags [F.], seq 1, ack 1, win 510, options [nop,nop,TS val 3783142158 ecr 3172660624], length 0
        0x0000:  4500 0066 b590 0000 4011 5059 c0a8 79f5  E..f....@.PY..y.
        0x0010:  c0a8 7957 95fb 2118 0052 0000 0800 0000  ..yW..!..R......
        0x0020:  0000 0400 5295 db2a 62ff 1ac5 deaf 6f54  ....R..*b.....oT
        0x0030:  0800 4500 0034 bc11 4000 4006 68aa 0a00  ..E..4..@.@.h...
        0x0040:  01e7 0a00 0022 1090 8e84 70ef ec7a e0b1  ....."....p..z..
        0x0050:  50a5 8011 01fe 71a7 0000 0101 080a e17e  P.....q........~
        0x0060:  2b0e bd1a f590                           +.....
19:35:49.613857 IP 192.168.121.87.33954 > 192.168.121.245.8472: OTV, flags [I] (0x08), overlay 0, instance 6
IP 10.0.0.34.36484 > 10.0.1.231.4240: Flags [.], ack 2, win 508, options [nop,nop,TS val 3172660625 ecr 3783142158], length 0
        0x0000:  4500 0066 ca12 0000 4011 3bd7 c0a8 7957  E..f....@.;...yW
        0x0010:  c0a8 79f5 84a2 2118 0052 0000 0800 0000  ..y...!..R......
        0x0020:  0000 0600 5a10 a0aa 2614 5a10 a0aa 2614  ....Z...&.Z...&.
        0x0030:  0800 4500 0034 8bf4 4000 4006 98c7 0a00  ..E..4..@.@.....
        0x0040:  0022 0a00 01e7 8e84 1090 e0b1 50a5 70ef  ."..........P.p.
        0x0050:  ec7b 8010 01fc 162f 0000 0101 080a bd1a  .{...../........
        0x0060:  f591 e17e 2b0e                           ...~+.
^C
3 packets captured
3 packets received by filter
0 packets dropped by kernel
root@node01:/home/cilium# 

# Here we are using `tcpdump`` to capture and display detailed network packets on the cilium_wg0 interface.
# The -n option avoids DNS lookups, and the -X option shows packet content in both hexadecimal and ASCII format.
# Via tcpdump, you should see the traffic between the pods.
# We see requests from curlpod to nginx and responses from nginx to curlpod in tcpdump output.
```

---
