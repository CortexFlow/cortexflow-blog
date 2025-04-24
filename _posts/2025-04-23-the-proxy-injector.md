---
layout: post
title:  "Service Mesh Explained: Building a Proxy Injector in Rust (with Code)"
description: "What is proxy injector? How an Admission Webhook works? Deep dive into the essential components for automating sidecar injection in Kubernetes."
author: lorenzo-tettamati 
categories: [ Cortexflow, service mesh,tutorial ]
image: "assets/images/the-proxy-injector.webp"
tags: [service mesh explained,featured,Rust,tutorial]
---
Kubernetes service meshes rely on **“sidecar”** proxies to handle traffic routing transparently, security 
policies, and observability for your microservices—but manually bolting those proxies onto every Pod 
spec quickly becomes a maintenance nightmare.

_What if you could have Kubernetes do the work for you, automatically injecting the proxy whenever a Pod is created?_ 

In this tutorial, we’re going to build exactly that: a **Mutating Admission Webhook** in Rust that hooks 
into the Kubernetes API server, inspects incoming Pod specs, and—if they meet your criteria—patches 
them on the fly to include an init‑container (for iptables setup) and your proxy‑sidecar.   

Along the way, you’ll learn how to: 

- Define the AdmissionReview/AdmissionRequest and AdmissionResponse data structures    
- Wire up an async handler in Axum, complete with #[instrument] tracing for per-request logging   
- Craft a JSONPatch that adds init‑containers and sidecar containers via a base64-encoded payload  
- Stand up a TLS‑secured HTTP server using Rustls so Kubernetes can trust your webhook   

By the end, you’ll have a drop‑in proxy injector that can be deployed alongside your service mesh 
control plane—no more manual injection, no more drift, just automatic, consistent proxy injection 
across your cluster. 

All the code we walk through here is available on our GitHub [repository](https://github.com/CortexFlow/CortexBrain)—feel free to clone and explore it!

Let’s dive in!🚀

## Admission Webhooks
_What Are Admission Webhooks?_  

Admission webhooks are a type of dynamic admission controller in Kubernetes. They allow you to validate or modify (mutate) Kubernetes objects as they are submitted to the cluster.

There are two types of admission webhooks:

- **Validating Admission Webhooks** – used to validate requests to the Kubernetes API server. They can accept or reject the request, but **cannot** modify the object.
- **Mutating Admission Webhooks** – used to **modify** (mutate) objects before they are persisted. They can change or enrich the resource definition, such as injecting sidecars into pods.

Admission webhooks are HTTP callbacks that are invoked during the admission phase of an API request. The Kubernetes API server sends an AdmissionReview request to the webhook service, which then evaluates the request and responds with an AdmissionReview response.

The admission phase takes place **after** authentication and authorization, but **before** the object is stored in etcd.

You can configure the Kubernetes API server to call specific webhook services when certain operations (like `CREATE`, `UPDATE`, or `DELETE`) are performed on specific resources (such as Pods, Deployments, etc.).

### How It Works

When a request is made to the Kubernetes API:

1. The request is authenticated and authorized.
2. The object goes through the admission phase, where it is passed to:
   - Mutating webhooks (in sequence),
   - Followed by validating webhooks (in parallel).
3. Based on the webhook responses, the request is either allowed, denied, or modified.
4. If allowed, the object is persisted in etcd.

### Controllers vs. Webhooks

It’s important to distinguish between **admission controllers** and **webhooks**:

- **Admission controllers** are built into the Kubernetes API server binary. They are enabled and configured by cluster administrators and cannot be extended at runtime.
- **Webhooks**, on the other hand, are **external HTTP services** configured through the Kubernetes API. They provide a more flexible and extensible way to implement custom admission logic, and can be written in any language or framework.

Admission controllers can validate, mutate, or perform both operations depending on their configuration. While validating controllers can **only inspect and accept/reject** objects, mutating controllers can **modify** them before they are stored.

## Building a Proxy Injector: The Structures

To begin, we need to define the data structures that will be used within our injector code. We use the `pub` keyword to make these structures accessible from other files within the module.
The first structure we need is `AdmissionRequest`, which represents a request sent to the admission webhook:

```
#[derive(Debug, Serialize, Deserialize)]
pub struct AdmissionRequest {
    uid: String,
    object: serde_json::Value,
}
```

- uid: A unique identifier for this admission request, provided by the Kubernetes API server. It's used to correlate requests and responses.
- object: This field contains the Kubernetes object (usually a Pod) being submitted. It's stored as a raw JSON value so we can inspect or mutate it flexibly.

Next, we define the AdmissionReview structure. This wraps the admission request and is used to process it:

```
#[derive(Debug, Deserialize, Serialize)]
pub struct AdmissionReview {
    #[serde(rename = "apiVersion", default = "default_api_version")]
    pub api_version: String,
    #[serde(default = "default_kind")]
    pub kind: String,
    pub request: AdmissionRequest,
    #[serde(skip_deserializing)]
    pub response: Option<AdmissionResponse>,
}
```

- api_version: The version of the AdmissionReview API we're handling.
- kind: Always "AdmissionReview" for admission webhooks.
- request: Contains the actual AdmissionRequest sent by the API server.
- response: Optional at deserialization time (we don't receive it from the client), but we populate it before responding to the API server.  
The default values for `apiVersion` and `kind` are provided by the following functions:

```
fn default_api_version() -> String {
    "admission.k8s.io/v1".to_string()
}

fn default_kind() -> String {
    "AdmissionReview".to_string()
}
```

After that, we define the AdmissionResponse structure, which is used to send a response back from the admission webhook:

```
#[derive(Debug, Serialize)]
pub struct AdmissionResponse {
    uid: String,
    allowed: bool,
    patch: Option<String>,
    #[serde(rename = "patchType")]
    patch_type: Option<String>,
}
```

- uid: Must match the request's UID so Kubernetes knows which request this response is for.
- allowed: Indicates whether the request is approved or denied.
- patch: If set, this is a base64-encoded JSON patch to modify the original object before it's persisted in etcd.
- patch_type: Typically "JSONPatch" if you're modifying the object. Required when patch is provided.

## Building a Proxy injector: The injection logic
After defining the main structures, we need to create the proper injection logic.  
We want a **modular logic** that can adapt to future changes and users' needs, while maintaining a **simple program structure**.

First of all, we need to create a simple function called `check_and_validate_pod`.  
This function ensures that the pod meets our requirements **before injecting the sidecar proxy** into a pod.

The function follows this logic:
- **Checks if containers are present**
  - Iterates over each container in the pod's `spec.containers`.
  - If a container's name contains `"cortexflow-proxy"`:
    - Logs an error.
    - Returns an error: "The pod is not eligible for proxy injection. Sidecar proxy already present."`

- **Validates namespace annotations**
  - Retrieves the pod's namespace from `metadata.namespace`.
  - Checks `metadata.annotations` for the key `"proxy-injection"`.
    - If it's set to `"disabled"`:
      - Logs a warning.
      - Returns an error: `"Automatic namespace injection is disabled."`

- **Validates pod-level annotations**
  - Checks if the pod itself has `"proxy-injection": "disabled"` in `metadata.annotations`.
    - If so:
      - Logs a warning.
      - Returns an error: `"Automatic pod injection is disabled."`

- **If all checks pass**
  - Returns `Ok(true)` indicating the pod is eligible for injection.

For the sake of brevity, I am not including the code below, but you can find the `check_and_validate_pod` code [here](https://github.com/CortexFlow/CortexBrain/blob/main/core/src/components/proxy-injector/src/validation.rs).

Going back to our inject function, after calling the validation function, we expect two behaviours:
1. The pod is ready and eligible for injection
2. The pod is not eligible for injection

In the first case, we can apply the patch, which we'll define in the next chapter, and return an `allowed: true` Admission Response.
In the second case, we are not injecting the patch, and we return an `allowed: false` Admission Response.

```
#[instrument]
pub async fn inject(
    State(_state): State<Arc<()>>,
    Json(mut admission_review): Json<AdmissionReview>,
) -> Json<AdmissionReview> {
    let pod = &admission_review.request.object;

    // Log the received pod metadata for debugging
    info!("Pod metadata: {:?}", pod["metadata"]);
    info!("Pod annotations: {:?}", pod["metadata"]["annotations"]);

    // Injection logic
    let response = if check_and_validate_pod(pod).unwrap_or(false) {
        info!("Starting the proxy injector");

        let patch_string = serde_json::to_string(&*PATCH).unwrap();
        let patch_encoded = STANDARD.encode(&patch_string);

        info!("Patch to be applied: {}", &patch_string);

        AdmissionResponse {
            uid: admission_review.request.uid.clone(),
            allowed: true,
            patch: Some(patch_encoded),
            patch_type: Some("JSONPatch".to_string()),
        }
    } else {
        AdmissionResponse {
            uid: admission_review.request.uid.clone(),
            allowed: false,
            patch: None,
            patch_type: None,
        }
    };

    admission_review.response = Some(response);
    admission_review.api_version = "admission.k8s.io/v1".to_string();
    admission_review.kind = "AdmissionReview".to_string();

    Json(admission_review)
}

```

## Building a Proxy injector: The patch
Now the magic happens ⭐. The patch is one of the most crucial parts in the proxy injector and is where all the variables are defined.  
We are using [`serde_json`](https://docs.rs/serde_json/latest/serde_json/) to create a JSON Patch and [`lazy_static`](https://docs.rs/lazy_static/latest/lazy_static/) to optimize the resources by initializing the variable when it is first accessed, in contrast to the regular static data, which is initialized at compile time.

The patch is divided into two parts:  
1. Initialize Iptables  
2. Initialize the proxy  

In the first part, we are using `iptables` to redirect all the external traffic — in particular, TCP and UDP traffic — to specific ports.  
We decided to bind the **TCP traffic** to port **5054** and the **UDP traffic** to port **5053**.

_**Note:**  
The `init-iptables` operation cannot be skipped. Otherwise, our system will not bind the traffic to the ports we chose, resulting in endless hours of debugging._

In the second part, we're doing another `add` operation to include the image of the proxy server.  
We're also explicitly setting the TCP (`5054`) and UDP (`5053`) ports using the `containerPort` key.

```
use lazy_static::lazy_static;
use serde_json::Value;

// Container patch
lazy_static! {
    pub static ref PATCH: Value = {
        let json_str = r#"[
            {
                "op": "add",
                "path": "/spec/initContainers",
                "value": [
                    {
                        "name": "init-iptables",
                        "image": "ubuntu",
                        "securityContext": {
                            "capabilities": {
                                "add": ["NET_ADMIN"]
                            }
                        },
                        "command": [
                            "/bin/sh", "-c",
                            "apt-get update && apt-get install -y iptables && iptables -t nat -A PREROUTING -p tcp -j REDIRECT --to-port 5054 && iptables -t nat -A PREROUTING -p udp -j REDIRECT --to-port 5053"
                        ]
                    }
                ]
            },
            {
                "op": "add",
                "path": "/spec/containers/-",
                "value": {
                    "name": "proxy-sidecar",
                    "image": "lorenzotettamanti/cortexflow-proxy:latest",
                    "ports": [
                        {
                            "containerPort": 5054,
                            "protocol": "TCP"
                        },
                        {
                            "containerPort": 5053,
                            "protocol": "UDP"
                        }
                    ]
                }
            }
        ]"#;
        serde_json::from_str(json_str).unwrap()
    };
}

```
## Building a Proxy injector: The server logic 
In the last part we need to create a server to serve the API we made in the previous step. For this step we use the axum crate and we proceed creating a route. We decided to call the endpoint `/mutate` as a reminder for our *Mutating Admission Webhook*. As second step we proceed to associate the inject function as POST request and we bind the 9443, this ends the route configuration. The last step is to load the *TLS* certificate files tls.crt and tls.key. 

_**Note:**
Kubernetes requires TLS certificates to serve APIs over HTTPS. Failing to provide the certificates will result in a non-functional webhook service_

_How to generate a TLS certificate?_  
Working with TLS certificates may be something unfamiliar to the majority of people reading this article. *Cert-manager* is the easiest way to generate the tls.key and tls.crt keys. All you have to do is installing cert-manager using the kubernetes CLI
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```
The installation may take a while so you can take a small break to let your mind rest a little bit!  
After cert-manager is installed you can get the secrets using the following commands
1. Return the data.ca file
```
kubectl get secret proxy-injector-tls -n cortexflow -o jsonpath='{.data.ca\.crt}'
```
2. Return the tls.key file
```
kubectl get secret proxy-injector-tls -n cortexflow -o jsonpath='{.data.tls\.key}'
```
3. Return the tls.crt file
```
kubectl get secret proxy-injector-tls -n cortexflow -o jsonpath='{.data.tls\.crt}'
```

_**Note:**
For security reasons, do not share these secrets with anyone. Leaking them may compromise your system’s security and get you in trouble._

We decided to automate this process in the install.sh script that you can find in the [repository](https://github.com/CortexFlow/CortexBrain/blob/58d97ca96cf79b82363c6553240e996409a667b0/Scripts/install.sh#L4) 

```
pub async fn run_server() -> Result<(), Box<dyn std::error::Error>> {
    let app = Router::new()
        .route("/mutate", post(inject))
        .with_state(Arc::new(()));

    let addr = SocketAddr::from(([0, 0, 0, 0], 9443));
    let config = RustlsConfig::from_pem_file(
        Path::new("/etc/webhook/certs/..data/tls.crt"),
        Path::new("/etc/webhook/certs/..data/tls.key"),
    )
    .await
    .map_err(|e| {
        error!("Failed to load TLS config: {}", e);
        e
    })?;

    info!("HTTPS server listening on {}", addr);

    if let Err(e) = axum_server::bind_rustls(addr, config)
        .serve(app.into_make_service())
        .await
    {
        error!("Server error: {}", e);
        return Err(Box::new(e));
    }

    Ok(())
}

```
## Building a Proxy injector: Deploying to Kubernetes

Now that all components are in place, the final step is to create a Kubernetes manifest to deploy the application into our cluster.
Below is an example YAML file we used to deploy the proxy injector within our namespace.

Pay special attention to the spec section: here, we define a custom selector that grants the necessary permissions for the injector to modify incoming Pod definitions—specifically, to add the sidecar proxy container automatically.
```
apiVersion: v1
kind: Secret
metadata:
  name: proxy-injector-tls
  namespace: cortexflow
type: kubernetes.io/tls
data:
  tls.crt: #omitted
  tls.key: #omitted
---
apiVersion: v1
kind: Service
metadata:
  name: proxy-injector
  namespace: cortexflow
spec:
  ports:
  - port: 443
    targetPort: 9443
  selector:
    app: proxy-injector
---
apiVersion: v1
kind: Pod
metadata:
  name: proxy-injector
  namespace: cortexflow
  labels:
    app: proxy-injector
spec:
  containers:
  - name: proxy-injector
    image: lorenzotettamanti/cortexflow-proxy-injector:latest
    ports:
    - containerPort: 9443
    volumeMounts:
    - name: webhook-certs
      mountPath: /etc/webhook/certs
      readOnly: true
  volumes:
  - name: webhook-certs
    secret:
      secretName: proxy-injector-tls
```

## Conclusion


In the first part, we've covered the foundamentals of proxy injection,going through admission webhooks and admission controllers, while in the second part we have built all the logic from scratch using the Rust programming covering a lot of practical aspects such as defining the structures,building the patch, launching the axum server and interacting with the Kubernetes API.


In the next part of this series, we’ll create a sidecar proxy and all the basic functions such as service discovery, metrics, observability and messaging 🚀

Enjoying the content? Show us some love with a ⭐ on [GitHub!](https://github.com/CortexFlow/CortexBrain) And be sure to catch the first episode of the series, where we take a deep dive into the world of service meshes.
**Stay tuned—and stay curious.** 🌐🧩