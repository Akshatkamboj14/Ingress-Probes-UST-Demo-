# Ingress-Probes-UST-Demo



**1. What is Ingress?**



**Definition**: Ingress is a Kubernetes object that manages external access to services in a cluster, typically for HTTP and HTTPS traffic. Think of it as the smart traffic cop for your cluster.



**Analogy**: Imagine your Kubernetes cluster is a big apartment building.



* Each apartment is a pod (your application).



* The apartment's lobby is a service (like a ClusterIP Service, which only works inside the building).



* The building's main entrance is an Ingress. It's the only way for visitors (external traffic) to get to the correct apartment (pod/service). You tell the doorman (Ingress Controller) which visitors go to which apartment.





