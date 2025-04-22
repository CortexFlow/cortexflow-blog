---
layout: post
title:  "Service Mesh Explained: The proxy injector (with code)"
description: "What is proxy injector? How an Admission Webhook works? Deep dive into the essential components for automating sidecar injection in Kubernetes."
author: lorenzo-tettamati 
categories: [ Cortexflow, service mesh,tutorial ]
image: "assets/images/the-proxy-injector.jpg"
tags: [service mesh explained,featured,Rust,tutorial]
---

Kubernetes service meshes rely on **“sidecar”** proxies to transparently handle traffic routing, security 
policies, and observability for your microservices—but manually bolting those proxies onto every Pod 
spec quickly becomes a maintenance nightmare.

_What if you could have Kubernetes do the work for you, automatically injecting the proxy whenever a Pod is created?_ 

In this tutorial, we’re going to build exactly that: a **Mutating Admission Webhook** in Rust that hooks 
into the Kubernetes API server, inspects incoming Pod specs, and—if they meet your criteria—patches 
them on the fly to include both an init‑container (for iptables setup) and your proxy‑sidecar.   

Along the way you’ll learn how to: 
- Define the AdmissionReview/AdmissionRequest and AdmissionResponse data structures (with 
Serde)   
- Wire up an async handler in Axum, complete with `#[instrument]` tracing for per-request logging   
- Craft a JSONPatch that adds init‑containers and sidecar containers via a base64-encoded payload   - Stand up a TLS‑secured gRPC/HTTP2 server using Rustls so Kubernetes can trust your webhook   

By the end, you’ll have a drop‑in proxy injector that can be deployed alongside your service mesh 
control plane—no more manual YAML editing, no more drift, just automatic, consistent proxy injection 
across your cluster. Let’s dive in!

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

## Building a Proxy injector: The Structures

To begin, we need to define the data structures that will be used within our injector code. We use the `pub` keyword to make these structures accessible from other files within the module.
The first structure we need is `AdmissionRequest`, which represents a request sent to the admission webhook:

```
#[derive(Debug, Serialize, Deserialize)]
pub struct AdmissionRequest {
    uid: String,
    object: serde_json::Value,
}
```
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
The default values for apiVersion and kind are provided by the following functions:
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
## Building a Proxy injector: The injection logic

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
