# Ingress-Probes-UST-Demo



**1. What is Ingress?**



**Definition**: Ingress is a Kubernetes object that manages external access to services in a cluster, typically for HTTP and HTTPS traffic. Think of it as the smart traffic cop for your cluster.



**Analogy**: Imagine your Kubernetes cluster is a big apartment building.



* Each apartment is a pod (your application).



* The apartment's lobby is a service (like a ClusterIP Service, which only works inside the building).



* The building's main entrance is an Ingress. It's the only way for visitors (external traffic) to get to the correct apartment (pod/service). You tell the doorman (Ingress Controller) which visitors go to which apartment.







                     \[ User / Client Browser ]

&nbsp;                          |

&nbsp;            Request: http://nodeIP:30080/app1

&nbsp;            Request: http://nodeIP:30080/app2

&nbsp;                          |

&nbsp;                          v

&nbsp;  ----------------------------------------------------------

&nbsp;  |              Kubernetes Cluster (Vanilla)              |

&nbsp;  |                                                        |

&nbsp;  |         \[ NodePort Service ] (port 30080)              |

&nbsp;  |   Exposes Ingress Controller to outside world          |

&nbsp;  |                           |                            |

&nbsp;  |                           v                            |

&nbsp;  |                 \[ Ingress Controller Pod ]             |

&nbsp;  |                    (e.g., NGINX)                       |

&nbsp;  |                           |                            |

&nbsp;  |             +-------------------------------+          |

&nbsp;  |             | Ingress Resource (Routing)    |          |  Rules from yaml -> take the cluster ip from api server through etcd, 

&nbsp;  |             |-------------------------------|          |

&nbsp;  |             | /app1 → Service: service-a    |          |

&nbsp;  |             | /app2 → Service: service-b    |          |

&nbsp;  |             +-------------------------------+          |

&nbsp;  |                           |                            |

&nbsp;  |          -----------------+-------------------          |

&nbsp;  |          |                                     |        |

&nbsp;  |          v                                     v        |

&nbsp;  |   \[ Service: service-a ]                \[ Service: service-b ] 

&nbsp;  |    ClusterIP, port 8080                  ClusterIP, port 9090 

&nbsp;  |          |                                     |        |

&nbsp;  |   +------+------+                      +-------+-------+ |

&nbsp;  |   | Pod A1 | Pod A2 |                  | Pod B1 | Pod B2 | |

&nbsp;  |   +--------+--------+                  +---------+--------+ |

&nbsp;  |                                                        |

&nbsp;  ----------------------------------------------------------



**Reverse Response Path:**  

**Pod → Service → Ingress Controller → NodePort (30080) → User**



* Users send requests to nodeIP:30080/app1 or /app2.



* That hits the NodePort, which forwards traffic to the Ingress Controller Pod.



* The Ingress resource checks routing rules:



* /app1 → service-a (ClusterIP → Pods A1/A2).



* /app2 → service-b (ClusterIP → Pods B1/B2).



Traffic flows back through the same path to the user.






🔹 On the Control Plane (Master Node)



These are Kubernetes brain components, they don’t handle real HTTP traffic but manage state:



1. API Server 🧩



Stores your Ingress YAML object when you do kubectl apply.



Exposes it via the Kubernetes API.



The Ingress Controller watches this.



2\. etcd 🗄️



Stores the actual Ingress resource definition and cluster state.



3\. Controller Manager ⚙️



Includes the Endpoint Controller, which updates Endpoints objects for Services used in Ingress rules.



4\. Scheduler



Decides where Pods (including the Ingress Controller Pod itself) should run.



👉 So, on the control plane, nothing is routing real requests — they are just making sure the Ingress object exists, is stored, and that mappings between Services ↔ Pods are correct.



🔹 On the Worker Plane (Nodes)



This is where the real data traffic flows:



1. Ingress Controller Pod (e.g., NGINX, HAProxy, Traefik)



Runs as a Deployment/DaemonSet in your cluster (on worker nodes).



Watches the API Server for Ingress objects.



Builds a routing table from Ingress rules.



Handles incoming external HTTP/HTTPS traffic.



2\. kube-proxy



Programs iptables/ipvs rules on the worker node.



Ensures that when the Ingress Controller forwards traffic to a Service IP, the packets reach the correct backend Pod.



3\. Pods (App Pods)



These are your actual application backends (e.g., frontend, API, database services).



They are the real destinations behind the Services the Ingress routes to.



4\. kubelet



Keeps track of Pods running on the worker node.



Reports Pod IPs and health to the API Server, which keeps Endpoints updated (used indirectly by Ingress).









**2. Challenges Before Ingress**

Before Ingress became a standard, developers faced significant problems when trying to expose their applications to the outside world.



**Challenge 1: Manual Port Management (NodePort)**



* **Problem**: With a NodePort service, every service you expose gets a unique port on every node in the cluster. For example, your web app might be on port 30080, and your API on 30081. This quickly becomes unmanageable.



* **Real-world issue:** It's hard to remember these arbitrary port numbers, and you can't use standard ports like 80 and 443 without special configuration.



**Challenge 2: Cost and Complexity (Load Balancers)**



* **Problem**: The LoadBalancer service type creates a dedicated, external cloud load balancer for every single service.



* **Real-world issue:** If you have 20 different microservices, you would need 20 separate cloud load balancers. This is incredibly expensive and difficult to manage. It also doesn't allow for smart routing based on the URL.



**Challenge 3: Lack of Intelligent Routing**



* **Problem:** Both NodePort and LoadBalancer services are very basic. They can only forward traffic to a single service. They can't inspect the incoming request to route it based on the URL or domain name.



* **Real-world issue:** You couldn't host api.example.com and blog.example.com on the same public IP address.



**3. How Ingress Solved These Challenges**



Ingress addressed the problems by creating a single, intelligent entry point for all external traffic.



* **Solution 1: Centralized Access on Standard Ports**



* Ingress provides a single public IP address that listens on standard ports 80 and 443. All external traffic comes through this single point, which is managed by a component called the Ingress Controller.



* **Solution 2: Cost-Effectiveness**



* You only need one external load balancer (or NodePort) to expose the single Ingress Controller. This significantly reduces costs compared to having a load balancer for every single service.



**Solution 3: Intelligent Routing**



* This is Ingress's superpower. It can inspect incoming requests and route them based on rules you define.





&nbsp;	\* Host-based routing: Traffic for api.example.com goes to one service, while traffic for blog.example.com goes to another.



&nbsp;	\* Path-based routing: Traffic for www.example.com/api goes to the API service, while www.example.com/shop goes to the store service.



**4. The Internal Working of Ingress**

This is the most important part to explain the mechanics. The Ingress system is made of two components that work together: the Ingress Resource and the Ingress Controller.



**a. The Ingress Resource (The Rules)**

This is the YAML object you create. It's just a set of instructions or rules.



**YAML breakdown:**



**YAML**



**apiVersion: networking.k8s.io/v1**

**kind: Ingress**

**metadata:**

  **name: example-ingress**

**spec:**

  **rules:**

  **- host: blog.example.com**

    **http:**

      **paths:**

      **- path: /**

        **pathType: Prefix**

        **backend:**

          **service:**

            **name: blog-service**

            **port:**

              **number: 80**

  **- host: api.example.com**

    **http:**

      **paths:**

      **- path: /v1**

        **pathType: Prefix**

        **backend:**

          **service:**

            **name: api-service**

            **port:**

              **number: 80**

You define rules based on the host (blog.example.com) and path (/v1).



The backend section tells the Ingress Controller which service to send the traffic to (blog-service).



b. The Ingress Controller (The Doorman)


* This is the actual program that runs as a pod in your cluster. It's the implementation of the rules.



* How it works:



&nbsp;	\* The Ingress Controller is constantly watching the Kubernetes API server for new or updated Ingress resources.



&nbsp;	\* When you create the YAML file above with kubectl apply, the API server stores it.



&nbsp;	\* The Ingress Controller sees the new resource and dynamically reconfigures itself.



&nbsp;	\* It sets up its own internal routing table based on the rules.



&nbsp;	\* When an external request comes in, the Ingress Controller looks at the request's Host header and URL path. It matches it against its routing table and forwards the request to the correct backend service.



**Role of NodePort and LoadBalancer:**



* To get traffic into the cluster, the Ingress Controller itself must be exposed.



* The most common way to do this is to use a LoadBalancer service for the Ingress Controller pod. The cloud provider automatically provisions a public IP and load balancer that directs all traffic to your Ingress Controller.



* If you don't have a cloud load balancer, you can use a NodePort service to expose the Ingress Controller on a static port on each node.



5\. Types of Ingress

It's important to differentiate between the API object and the implementation.



Ingress Resource: This is the standard API object. It's the same regardless of the controller you use.



Ingress Controller: This is the actual software that implements the Ingress rules. Popular options include:



* NGINX Ingress Controller: One of the most popular and widely used. It's a very performant and feature-rich choice.



* Traefik: Known for its easy configuration and dynamic updates.







**How Ingress is Used in Production**


In a real-world production environment, Ingress solves several key challenges:



* Centralized Traffic Management: Instead of managing multiple public IP addresses and DNS records for each microservice, a company uses a single public IP exposed by an Ingress controller. This simplifies networking, reduces costs, and makes management of the system much easier. For example, a single Ingress controller can manage all incoming traffic to a complex application with a dozen microservices.





* Host-Based and Path-Based Routing: In a production setting, an Ingress controller is used for intelligent routing.



&nbsp;	- Host-Based Routing: A company's main website, its admin portal, and its customer-facing API can all be hosted on the same cluster using a single 	Ingress. The Ingress rules would route www.example.com to the front-end service, admin.example.com to the internal admin portal, and api.example.com 	to the API gateway service.



&nbsp;	- Path-Based Routing: For a single domain, Ingress can direct traffic to different services based on the URL path. For instance, 	www.example.com/shop could go to the Shopping service, while www.example.com/blog could be routed to the Blog service.



* SSL/TLS Termination: Ingress handles all SSL encryption and decryption at the cluster's edge. This means your application pods don't have to manage SSL certificates or perform the intensive cryptographic work. You can use tools like Cert-Manager with Ingress to automatically issue and renew certificates from services like Let's Encrypt, ensuring all traffic is secure without manual intervention.







* Blue-Green or Canary Deployments: Ingress can be configured for advanced deployment strategies. For a blue-green deployment, you can run two versions of your application simultaneously. The Ingress can instantly switch all traffic from the old "blue" version to the new "green" version with a simple change in the routing rule, ensuring zero downtime. For a canary deployment, you can route a small percentage of traffic (e.g., 5%) to a new version of your service to test it in a live environment before fully rolling it out.





Real-World Example: An E-commerce Platform


* Consider a large e-commerce platform that has been broken down into multiple microservices.



* Customer-facing website: This is a front-end application with its own pod and service.



* Product Catalog service: Manages all product listings and images.



* User Authentication service: Handles user logins and sessions.



* Shopping Cart service: Manages the user's shopping cart.



* Payment Processing service: Handles all transactions.



In this scenario, a single Ingress controller would manage all external traffic. A user's request would first hit the Ingress, which would then route the request to the correct service based on the URL. For example, a request to store.com/login would be routed to the User Authentication service, while a request to store.com/cart would go to the Shopping Cart service.







**Probes**

probes are checks performed by the kubelet on containers in a Pod to determine their health and readiness. They are defined in the Pod specification under a container.


**Why We Use Probes**


The primary reason for using probes is to enable self-healing and high availability in your applications. Without probes, Kubernetes only knows if a container is running or not. It can't tell if the application inside the container has frozen, is stuck in a deadlock, or is otherwise unable to serve traffic. Probes provide a mechanism for your application to report its health to Kubernetes, which then acts on that information.

**Benefits of Using Probes**


**Improved Reliability:** By detecting and restarting unhealthy containers, probes prevent your service from becoming unavailable.



**Zero Downtime Deployments:** Probes help Kubernetes orchestrate rolling updates smoothly. It won't send traffic to a new pod until its readiness probe passes, ensuring the new version is ready to serve requests before the old one is terminated.



**Resource Efficiency:** Probes can prevent wasted resources. If a container is not healthy, Kubernetes can restart it, potentially freeing up resources that the unhealthy process was holding.



**Types of Probes**


**There are three main types of probes in Kubernetes:**



**Liveness Probe:** This probe determines if a container is alive and running. If a liveness probe fails, Kubernetes assumes the container is in a failed state and will restart it.



**apiVersion: v1**

**kind: Pod**

**metadata:**

  **name: my-app-pod**

**spec:**

  **containers:**

    **- name: my-app**

      **image: my-app:latest**

      **livenessProbe:**

        **httpGet:**

          **path: /healthz**

          **port: 8080**

        **initialDelaySeconds: 15**

        **periodSeconds: 20**






**Readiness Probe:** This probe indicates if a container is ready to serve traffic. If a readiness probe fails, Kubernetes will remove the pod's IP address from the service endpoints. This means no traffic will be routed to that pod until the probe starts succeeding again. Unlike the liveness probe, a failed readiness probe does not cause a container to be restarted.



**apiVersion: apps/v1**

**kind: Deployment**

**metadata:**

  **name: my-web-server**

**spec:**

  **replicas: 3**

  **selector:**

    **matchLabels:**

      **app: webserver**

  **template:**

    **metadata:**

      **labels:**

        **app: webserver**

    **spec:**

      **containers:**

      **- name: my-web-server-container**

        **image: nginx:latest**

        **ports:**

        **- containerPort: 80**

        **readinessProbe:**

          **httpGet:**

            **path: /healthz**

            **port: 80**

          **initialDelaySeconds: 5**

          **periodSeconds: 10**







**Startup Probe:** This probe is used for applications that have a long startup time. If a startup probe is configured, all other probes (liveness and readiness) are temporarily disabled until the startup probe succeeds. This prevents the liveness probe from prematurely restarting a container that is still initializing. Once the startup probe passes, the normal liveness and readiness checks begin.



**apiVersion: apps/v1**

**kind: Deployment**

**metadata:**

  **name: slow-app**

**spec:**

  **replicas: 1**

  **selector:**

    **matchLabels:**

      **app: slow-app**

  **template:**

    **metadata:**

      **labels:**

        **app: slow-app**

    **spec:**

      **containers:**

      **- name: slow-app-container**

        **image: my-slow-app:latest**

        **startupProbe:**

          **exec:**

            **command:**

            **- /bin/check-db-connection.sh**

          **initialDelaySeconds: 5**

          **failureThreshold: 30**

          **periodSeconds: 10**

        **livenessProbe:**

          **httpGet:**

            **path: /healthz**

            **port: 8080**

          **initialDelaySeconds: 100**

          **periodSeconds: 10**





**How to Use Probes**


You define probes in the spec.containers section of your Kubernetes YAML manifest. There are three main types of handlers for probes:



* HTTP GET: Kubernetes sends an HTTP GET request to a specified path and port on the container. The probe is successful if the HTTP response code is in the 200-399 range. This is the most common method for web applications.



* TCP Socket: Kubernetes tries to open a TCP connection to a specified port on the container. The probe succeeds if the connection is established. This is useful for backend services that don't have an HTTP endpoint.



* Exec: Kubernetes executes a command inside the container. The probe is successful if the command exits with a status code of 0. This is a very flexible option for running custom health scripts.
  



**Best Practices**





* Keep Probes Lightweight: Your health check endpoint should be simple and fast. It should not perform heavy or time-consuming operations like complex database queries. A probe that takes too long can make Kubernetes incorrectly think your application is unhealthy.



* Use liveness and readiness probes together: They serve different purposes. A readiness probe ensures no traffic is sent to a pod until it's ready, which is crucial for zero-downtime deployments. A liveness probe ensures a broken application process is restarted.



* Use startup probes for slow-starting apps: If your application takes a long time to initialize, use a startup probe. This prevents the liveness probe from prematurely killing the container before it has a chance to start, which can lead to a continuous restart loop.



* Set initialDelaySeconds: This parameter tells Kubernetes to wait a certain number of seconds after the container starts before performing the first probe. This gives your application a little time to initialize before being checked.



* Set Realistic timeoutSeconds: This is the time Kubernetes waits for a probe to return a response. Set this value high enough to account for temporary slowdowns, but low enough to detect a truly unresponsive application.



* Choose the right probe type: An HTTP GET probe is great for web servers. A TCP Socket probe is good for non-HTTP services. An Exec probe is best for custom logic and legacy applications.



* Monitor your probes: Regularly check logs and metrics for probe failures to understand why your application might be failing its health checks. This data is invaluable for debugging and improving your application's stability.



  






