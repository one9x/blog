---
layout: post
title: kubernetes calico policy
date: 2020-12-24T06:36:29.609Z
author: Raman
categories: "Kubernetes "
---
<!--StartFragment-->

Learn Kubernetes Calico Policy

### Concepts

The Kubernetes Network Policy API provides a standard way for users to define network policy for controlling network traffic. However, Kubernetes has no built-in capability to enforce the network policy. To enforce network policy, you must use a network plugin such as Calico.

#### Ingress and egress

The bulk of securing network traffic typically revolves around defining egress and ingress rules. From the point of view of a Kubernetes pod, ingress is incoming traffic to the pod, and egress is outgoing traffic from the pod. In Kubernetes network policy, you create ingress and egress “allow” rules independently (egress, ingress, or both).

#### Default deny/allow behavior

Default allow means all traffic is allowed by default, unless otherwise specified. Default deny means all traffic is denied by default, unless explicitly allowed.

#### Create ingress policies

Create ingress network policies to allow inbound traffic from other pods.

Network policies apply to pods within a specific namespace. Policies can include one or more ingress rules. To specify which pods in the namespace the network policy applies to, use a pod selector. Within the ingress rule, use another pod selector to define which pods allow incoming traffic, and the ports field to define on which ports traffic is allowed.

##### Allow ingress traffic from pods in the same namespace

In the following example, incoming traffic to pods with label color=blue are allowed only if they come from a pod with color=red, on port 80.



kind: NetworkPolicy

apiVersion: networking.k8s.io/v1

metadata:

  name: allow-same-namespace

  namespace: default

spec:

  podSelector:

    matchLabels:

      color: blue

  ingress:

\- from:

\- podSelector:

        matchLabels:

          color: red

    ports:

\- port: 80

 

##### Allow ingress traffic from pods in a different namespace

To allow traffic from pods in a different namespace, use a namespace selector in the ingress policy rule. In the following policy, the namespace selector matches one or more Kubernetes namespaces and is combined with the pod selector that selects pods within those namespaces.

Note: Namespace selectors can be used only in policy rules. The spec.podSelector applies to pods only in the same namespace as the policy.

In the following example, incoming traffic is allowed only if they come from a pod with label color=red, in a namespace with label shape=square, on port 80.

kind: NetworkPolicy

apiVersion: networking.k8s.io/v1

metadata:

  name: allow-same-namespace

  namespace: default

spec:

  podSelector:

    matchLabels:

      color: blue

  ingress:

\- from:

\- podSelector:

        matchLabels:

          color: red

      namespaceSelector:

        matchLabels:

          shape: square

    ports:

\- port: 80

 

#### Create egress policies

Create egress network policies to allow outbound traffic from pods.

##### Allow egress traffic from pods in the same namespace

The following policy allows pod outbound traffic to other pods in the same namespace that match the pod selector. In the following example, outbound traffic is allowed only if they go to a pod with label color=red, on port 80.

kind: NetworkPolicy

apiVersion: networking.k8s.io/v1

metadata:

  name: allow-egress-same-namespace

  namespace: default

spec:

  podSelector:

    matchLabels:

      color: blue

  egress:

\- to:

\- podSelector:

        matchLabels:

          color: red

    ports:

\- port: 80

 

##### Allow egress traffic to IP addresses or CIDR range

Egress policies can also be used to allow traffic to specific IP addresses and CIDR ranges. Typically, IP addresses/ranges are used to handle traffic that is external to the cluster for static resources or subnets.

The following policy allows egress traffic to pods in CIDR, 172.18.0.0/24.

kind: NetworkPolicy

apiVersion: networking.k8s.io/v1

metadata:

  name: allow-egress-external

  namespace: default

spec:

  podSelector:

    matchLabels:

      color: red

  egress:

\- to:

\- ipBlock:

        cidr: 172.18.0.0/24

 

#### Best practice: create deny-all default network policy

To ensure that all pods in the namespace are secure, a best practice is to create a default network policy. This avoids accidentally exposing an app or version that doesn’t have policy defined.

##### Create deny-all default ingress and egress network policy

The following network policy implements a default deny-all ingress and egress policy, which prevents all traffic to/from pods in the policy-demo namespace. Note that the policy applies to all pods in the policy-demo namespace, but does not explicitly allow any traffic. All pods are selected, but because the default changes when pods are selected by a network policy, the result is: deny all ingress and egress traffic. (Unless the traffic is allowed by another network policy).

kind: NetworkPolicy

apiVersion: networking.k8s.io/v1

metadata:

  name: default-deny

  namespace: policy-demo

spec:

  podSelector:

    matchLabels: {}

  policyTypes:

\- Ingress

\- Egress

 

### Above and beyond

* [Kubernetes Network Policy API documentation](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13/#networkpolicy-v1-networking-k8s-io)
* ![Calico Enterprise](https://lh4.googleusercontent.com/l2GjFWTMaesoAUoN8koDz8l3nof1Uo5scGYX5WYe-uNqaFruL0yMT38nRjPVr13dlrOWyTdYWOSnqFdiLm2UzvOS9hIDbzRdj0CELusnKkZmIYyowidQQQ96oI004KtD1iLDiv_M)[Compliance and threat detection with Calico Enterprise](https://docs.projectcalico.org/security/calico-enterprise/network-visibility)

[ Slack](https://slack.projectcalico.org/) [ Discourse](https://discuss.projectcalico.org/) [ GitHub](https://github.com/projectcalico/calico) [ Twitter](https://twitter.com/projectcalico) [ YouTube](https://www.youtube.com/channel/UCFpTnXDNcBoXI4gqCDmegFA) [ Training](https://www.tigera.io/tigera-products/calico-essentials/)

* [Big picture](https://docs.projectcalico.org/security/kubernetes-network-policy#big-picture)
* [Value](https://docs.projectcalico.org/security/kubernetes-network-policy#value)
* [Features](https://docs.projectcalico.org/security/kubernetes-network-policy#features)
* [Concepts](https://docs.projectcalico.org/security/kubernetes-network-policy#concepts)
* * [Ingress and egress](https://docs.projectcalico.org/security/kubernetes-network-policy#ingress-and-egress)
  * [Default deny/allow behavior](https://docs.projectcalico.org/security/kubernetes-network-policy#default-denyallow-behavior)
* [How to](https://docs.projectcalico.org/security/kubernetes-network-policy#how-to)
* * [Create ingress policies](https://docs.projectcalico.org/security/kubernetes-network-policy#create-ingress-policies)
  * * [Allow ingress traffic from pods in the ](https://docs.projectcalico.org/security/kubernetes-network-policy#allow-ingress-traffic-from-pods-in-the-same-namespace)

<!--EndFragment-->