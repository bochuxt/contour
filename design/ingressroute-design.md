# Executive Summary

**Status**: _Draft_

This document describes the design of a new CRD to replace the v1beta1.Ingress Kubernetes object.
This new CRD will be integrated into Contour 0.5/0.6.
Contour will continue to support the current v1beta1.Ingress object for as long as it is supported in upstream Kubernetes.

# Goals

- Support multi-tenant clusters, with the ability to limit the management of routes to specific virtual hosts.
- Support delegating the configuration of some or all the routes for a virtual host to another Namespace
- Create a clear separation between singleton configuration items--items that apply to the virtual host--and sets of configuration items--that is, routes on the virtual host.

# Non-goals

- Support Ingress objects outside the current cluster
- IP in IP or GRE tunneling between clusters with discontinuous or overlapping IP ranges.
- Support non HTTP/HTTPS traffic ingress--that is, no UDP, no MongoDB, no TCP passthrough. HTTP/HTTPS ingress only.
- Deprecate Contour support for v1beta1.Ingress.

# Background

The Ingress object was added to Kubernetes in version 1.2 to describe properties of a cluster-wide reverse HTTP proxy.
Since that time, the Ingress object has not progressed beyond the beta stage, and its stagnation inspired an explosion of annotations to express missing properties of HTTP routing.

# High-level design

At a high level, this document proposes modeling ingress configuration as a graph of documents throughout a Kubernetes API server, which when taken together form a directed acyclic graph (DAG) of the configuration for virtual hosts and their constituent routes.

## Delegation

The working model for delegation is DNS. 
As the owner of a DNS domain, for example `.com`, I _delegate_ to another nameserver the responsibility for handing the subdomain `heptio.com`.
Any nameserver can hold a record for `heptio.com`, but without the linkage from the parent `.com` TLD, its information is unreachable and non authoritative.

Each _root_ of a DAG starts at a virtual host, which describes properties such as the fully qualified name of the virtual host, any aliases of the vhost (for example, a `www.` prefix), TLS configuration, and possibly global access list details.
The vertices of a graph do not contain virtual host information. They are reachable from a root only by delegation.
This permits the _owner_ of an ingress root to both delegate the authority to publish a service on a portion of the route space inside a virtual host, and to further delegate authority to publish and delegate.

In practice the linkage, or delegation, from root to vertex, is performed with a specific type of route action.
You can think of it as routing traffic to another ingress route for further processing, instead of routing traffic directly to a service.

# Detailed Design

## IngressRoute CRD

This is an example of a fully populated root ingress route.

```yaml
apiVersion: contour.heptio.com/v1alpha1
kind: IngressRoute
metadata:
  name: google
  namespace: prod
spec:
  # virtualhost appears at most once. If it is present, the object is considered
  # to be a "root".
  virtualhost:
    # the fully qualified domain name of the root of the ingress tree
    # all leaves of the DAG rooted at this object relate to the fqdn (and its aliases)
    fqdn: www.google.com
    # a set of aliases for the domain, these may be alternative fqdns which are considered
    # aliases of the primary fqdn
    aliases:
      - www.google.com.au
      - google.com
    # if present describes tls properties. The CNI names that will be matched on
    # are described in fqdn and aliases, the tls.secretName secret must contain a
    # matching certificate
    tls:
      # required, the name of a secret in the current namespace
      secretName: google-tls
      # other properties like cipher suites may be added later
  strategy: RoundRobin # (Optional) LB Algorithm to apply to all services, defaults for all services
  lbHealthCheck (Optional):
    path: /healthz # HTTP endpoint used to perform health checks on upstream service (e.g. /healthz). It expects a 200 response if the host is healthy. The upstream host can return 503 if it wants to immediately notify downstream hosts to no longer forward traffic to it.
    intervalSeconds: 30 # The interval (seconds) between health checks. Defaults to 5 seconds if not set.
    timeoutSeconds: 60 # The time to wait (seconds) for a health check response. If the timeout is reached the health check attempt will be considered a failure. Defaults to 2 seconds if not set.
    unhealthyThresholdCount: 3 # The number of unhealthy health checks required before a host is marked unhealthy. Note that for http health checking if a host responds with 503 this threshold is ignored and the host is considered unhealthy immediately. Defaults to 3 if not defined.
    healthyThresholdCount: 5 # The number of healthy health checks required before a host is marked healthy. Note that during startup, only a single successful health check is required to mark a host healthy
  # routes contains the set of routes for this virtual host.
  # routes must _always_ be present and non empty.
  # routes can be present in any order, and will be matched from most to least specific.
  routes:
  # each route entry starts with a prefix match
  # and one of service or delegate
  - match: /static
    # service defines the properties of the service in the current namespace
    # that will handle traffic matching the route
    service:
    - name: google-static
      port: 9000
  - match: /finance
    # delegate delegates the matching route to another IngressRoute object.
    # This delegates the responsibility for /finance to the IngressRoute matching the delegate parameters
    delegate:
      # the name of the IngressRoute object to delegate to
      name: google-finance
      # the namespace of the IngressRoute object, if blank the current namespace is assumed
      namespace: finance
  - match: /ads
    # more than one service name/port pair may be present allowing the services for a route to
    # be handled by more than one service.
    service:
    - name: ads-red
      port: 9090
      weight: 50 # (Optional) Weight distribution across multiple services (If not defined then even distribution is assumed)
      strategy: RoundRobin # (Optional) LB Algorithm to apply to service (Defaults to RoundRobin)
    - name: ads-blue
      port: 9090
      weight: 50
      lbHealthCheck (Optional):
        path: /healthz # HTTP endpoint used to perform health checks on upstream service (e.g. /healthz). It expects a 200 response if the host is healthy. The upstream host can return 503 if it wants to immediately notify downstream hosts to no longer forward traffic to it.
        intervalSeconds: 30 # The interval (seconds) between health checks. Defaults to 5 seconds if not set.
        timeoutSeconds: 60 # The time to wait (seconds) for a health check response. If the timeout is reached the health check attempt will be considered a failure. Defaults to 2 seconds if not set.
        unhealthyThresholdCount: 3 # The number of unhealthy health checks required before a host is marked unhealthy. Note that for http health checking if a host responds with 503 this threshold is ignored and the host is considered unhealthy immediately. Defaults to 3 if not defined.
        healthyThresholdCount: 5 # The number of healthy health checks required before a host is marked healthy. Note that during startup, only a single successful health check is required to mark a host healthy.

```
This is an example of the `google-finance` object which has been delegated responsibility for paths starting with `/finance`.

```yaml
apiVersion: contour.heptio.com/v1alpha1
kind: IngressRoute
metadata:
  name: google-finance
  namespace: finance
spec:
  # note that this is a vertex, so there is no virtualhost key
  # routes contains the set of routes for this virtual host.
  # routes must _always_ be present and non empty.
  # routes can be present in any order, and will be matched from most to least
  # specific, prefixes defined here are appended to those delegated to this object.
  routes:
  # each route in this object is appended to the prefix which delegated to this object, so any path displayed here will be appended to the root
  - match: /
    service:
    - name: finance-app
      port: 9999
  - match: /static # Full path is: www.google.com/finance/static (also all aliases)
    service:
    - name: finance-static-content
      port: 8080
  - match: /partners
    # delegate the /partners prefix to the partners namespace
    delegate:
      name: finance-partners
      namespace: partners # www.google.com/finance/partners is delegated (also aliases)
```

### Load Balancing

Each upstream service can have a load balancing strategy applied to determine which host is selected for the request. 
The following list are the options available to choose from: 

- **RoundRobin:** Each healthy upstream host is selected in round robin order
- **WeightedLeastRequest:** The least request load balancer uses an O(1) algorithm which selects two random healthy hosts and picks the host which has fewer active requests. _Note: This algorithm is simple and sufficient for load testing. It should not be used where true weighted least request behavior is desired_
- **Ring hash:** The ring/modulo hash load balancer implements consistent hashing to upstream hosts
- **Maglev:** The Maglev load balancer implements consistent hashing to upstream hosts
- **Random:** The random load balancer selects a random healthy host

More documentation on Envoy's lb support can be found here: [https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/load_balancing.html](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/load_balancing.html)

### Healthcheck

Active health checking can be configured on a per upstream cluster basis. 
Contour will only support HTTP health checking along with various settings (check interval, failures required before marking a host unhealthy, successes required before marking a host healthy, etc.). 
During HTTP health checking Envoy will send an HTTP request to the upstream host. 
It expects a 200 response if the host is healthy. 
The upstream host can return 503 if it wants to immediately notify downstream hosts to no longer forward traffic to it.

_Note: Passive health checking is implemented via Outlier detection and is used to dynamically determine whether some number of hosts in an upstream cluster are performing unlike others and removing them from the healthy load balancing set. Passive checking is not included yet but will be in a future release._

## Delegation rules

The delegation rules applied are as follows

1. If an `IngressRoute` object contains a `spec.virtualhost` key it is considered a root.
2. If an `IngressRoute` object does not contain `spec.virtualhost` key is considered a vertex.
3. A vertex is reachable if a delegation to it exists in another `IngressRoute` object.
4. Vertices which are not reachable are considered orphaned. Orphaned vertices have no effect on the running configuration.

## Validation rules

The validation rules applied in this design are as follows. 
Some of these rules may be relaxed in future designs.

If a validation error is encountered, the entire object is rejected. Partial application of the valid portions of the configuration is **not** attempted.


## Authorization

It is important to highlight that both root and vertex IngressRoute objects are of the same type.
This is a departure from other designs which treat the permission to create a VirtualHost type object and a Route type object as separate. 
The DAG design treats the delegation from one IngressRoute to another as permission to create routes.

### Enforcing Mode

While the IngressRoute delegation allows for Administrators to limit route usage by namespace, it does not restrict where the `root` IngressRoutes can be created. 
Contour should allow for an `enforcing` mode which takes in a set of namespaces where root IngressRoutes are valid.
Only those permitted to operate in those namespaces can therefore create virtual hosts and delegate the permission to operate on them to other namespaces. 
This would most likely be accomplished with a command line flag (`--root-namespaces=[]`) or ConfigMap.

## Reporting status

Status about the object should be reported by using a scheme within the IngressRoute named `status`. There are a few different statuses that can reported. 

### Orphaned Status

The presence of a semantically valid object is not a guarantee that it will be used.
This is a break from the Kubernetes model, in which a valid object is generally be acted on by controllers.

An `IngressRoute` vertex may be present but not consulted if it is not part of a delegation chain from a root.
This models the DNS model above. You can add any zone file that you want to your local DNS server, but unless someone delegates to you, those records are ignored.

In the case where an IngressRoute is present, but has no active delegation, it is known as **orphaned**.
We record this information in a top level `status` key for operators and tools.

An example of an orphaned IngressRoute object:

```yaml
status:
  delegationStatuses:
  - state:
     orphaned:
       lastParent:
         name: ...
         namespace: ...
```

```yaml
status:
  orphaned: true
```

### Delegation Status

Since delegate IngressRoutes do not contain the VHost or Path information, it's important to understand what root IngressRoutes are delegating to a namespace.
This will be accomplished by reporting a `valid` status and include the root IngressRoute information.

An example of a valid IngressRoute object:

```yaml
status:
  delegationStatuses:
  - state:
      valid:
        roots: ...
        - name: ...
          namespace: ...
          - fqdn: ...
            aliases:
              - ...
              - ...
```

New Design:
```yaml
status:
  delegationStatuses:
  - www_google_com/path/to/my/ingressRoute # <-------- Validate this!
    status: 
      currentStatus: Current Status of the route (e.g. Invalid, Error, <needs defined>)
      lastProcessTime: Time the object was processed
      []errors: List of errors
```

### 

Note: `delegationStatuses` is a list, because in the non-orphaned case, an IngressRoute may be a part of several delegation chains.

An example of a correctly delegated IngressRoute with a single parent:
```yaml:
status:
  delegationStatuses:
  - state:
      connected:
        parent:
          name: ...
          namespace: ...
          prefix: ...
        connectedAt: ...
```

## IngressRoutes dispatch only to services in the same namespace

While a matching route may list more than one service/port pair, the services are always within the same namespace.
This is to prevent unintentional exposure of services running in other namespaces.

To publish a service in another namespace on a domain that you control, you must delegate the permission to publish to the namespace.
For example, the `kube-system/heptio` IngressRoute holds information about `heptio.com` but delegates the provision of the service to the `heptio-wordpress/wordpress` IngressRoute:

```yaml
apiVersion: contour.heptio.com/v1alpha1
kind: IngressRoute
metadata:
  name: heptio 
  namespace: kube-system
spec:
  virtualhost:
    fqdn: heptio.com
    aliases:
      - www.heptio.com
  routes:
  - match: /
    # delegate everything to heptio-wordpress/wordpress
    delegate:
      name: wordpress
      namespace: heptio-wordpress
---
apiVersion: contour.heptio.com/v1alpha1
kind: IngressRoute
metadata:
  name: wordpress 
  namespace: heptio-wordpress
spec:
  routes:
  - match: /
    service:
      name: wordpress-svc
      port: 8000
```

## TLS

TLS configuration, certificates, and cipher suites, remain similar in form to the existing Ingress object.
However, because the `spec.virtualhost.tls` is present only in root objects, there is no ambiguity as to which IngressRoute holds the canonical TLS information. (This cannot be said for the current Ingress object).
This also implies that the IngressRoute root and the TLS Secret must live in the same namespace.
However as mentioned above, the entire routespace (/ onwards) can be delegated to another namespace, which allows operators to define virtual hosts and their TLS configuration in one namespace, and delegate the operation of those virtual hosts to another namespace.

# Example Use-Cases

## Blue-Green deployments

Blue-green deployment is a technique that reduces downtime and risk by running two identical production environments called Blue and Green.
This can be accomplished by creating two delegated Ingress resources, one for Blue and one for Green. 
The Green IngressRoute would essentially be an Orphan route, then the user would swap the delegate IngressResource from pointing from the Blue to the Green so that traffic switches. 

## Canary Deployments

Similar to how Blue/Green deployments swap traffic between different versions of an application, Canary can also be an effective alternative.
This is accomplished by moving traffic into a new deployment slowly so that the new service can be watched for performance or any other kind of errors.
To implement Canary deployments, the user creates a new version of an application along side the existing, utilizing `weights` on the backend services, traffic can be slowly moved between the services.

_Note: This pattern will only work if your application can be run as two different versions at the same time._

# Alternatives considered

This proposal is presented as an alternative to more traditional designs that split ingress into two classes of object: one describing the properties of the ingress virtual host, the other describing the properties of the routes attached to that virtual host.

The downside of these proposals is that the minimum use case, the http://hello.world case, needs two CRD objects per ingress: one stating the url of the virtual host, the other stating the name of the CRD that holds the url of the virtual host.

Restricting who can create this pair of CRD objects further complicates things and moves toward a design with a third and possibly fourth CRD to apply policy to their counterparts.

Overloading, or making the CRD polymorphic, creates a more complex mental model for complicated deployments, but in exchange scales down to a single CRD containing both the vhost details and the route details, because there is no delegation in the hello world example.
This property makes the proposed design appealing from a usability standpoint, as **most** ingress use cases are simple--publish my web app on this URL--so it feels right to favour a design that does not penalize the default, simple, use case.

- links to other proposals

# Future work

Future work outside the scope of this design includes:

- Delegation to matching labels, rather than names. This may be added in the future. This is valid, as long as none of the matching IngressRoute objects are roots, because routes are a set, so can be merged from several objects _in the same namespace_.
- Allowing users to send traffic multiple upstreams by weighting first, or by lbAlgorithm first (e.g. Create a set of endpoints from multiple services to send traffic).
- Outlier detection for passive health checking (https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/outlier#arch-overview-outlier-detection).

# Security Concerns

As it relates to the CRD, the security implications of this proposal, the model of delegating a part of a vhost's route space to a CRD in another namespace implies a level of trust.
I, the owner of the virtual host, delegate to you the /foo prefix and everything under it, implies that you cannot operate on this vhost outside /foo, but it also means that whatever you do in /foo--host malware, leak k8s session keys--is something I'm trusting you not to do.

## No ability to prevent route delegation

In writing up this proposal an issue occurred to me.
It happens when Contour is operating in the poorly specified enforcing mode, where Contour recognizes root IngressRoutes only in a set of authorized namespaces.
(In fact it happens regardless of enforcement mode, but the argument is that if you do not turn on enforcement you are explicitly saying you trust you users, or you have another mechanism in place -- CI/CD -- to handle this).

Imaging this scenario:
```yaml
apiVersion: contour.heptio.com/v1alpha1
kind: IngressRoute
metadata:
  # this is www.google.com's root object
  name: google
  namespace: virtualhost-namespace
spec:
  virtualhost:
    fqdn: www.google.com
    tls:
      # required, the name of a secret in the current namespace
      secretName: google-tls
      # other properties like cipher suites may be added later
  routes:
  - match: /mail
    # delegate to the gmail service running in another namespace
    delegate:
      name: gmail
      namespace: gmail
    ...
```
And in the gmail namespace we have a IngressRoute vertex
```yaml
apiVersion: contour.heptio.com/v1alpha1
kind: IngressRoute
metadata:
  name: gmail 
  namespace: gmail
spec:
  routes:
  - match: /mail
    service:
      name: gmail-tomcat-svc
      port: 8000
```
That's all fine, the operators in the `gmail` namespace can't change the delegation in the `virtualhost-namespace` and the admins in the `virtualhost-namespace` feel confident that they control `www.google.com` and that gmail is being served over TLS.

However, at some time later ad money is getting tight and the operators take on a new customer, who is very cheap and doesn't want to pay for TLS.
```yaml
apiVersion: contour.heptio.com/v1alpha1
kind: IngressRoute
metadata:
  name: daves-cheap-webhosting
  namespace: virtualhost-namespace
spec:
  virtualhost:
    fqdn: dave.cheney.net
  routes:
  - match: /
    delegate:
      name: webapp
      namespace: customer-dave
```
Because the ingress operators don't want to have to handle tickets for dave's cheap webhosting, they delegate `/` to dave's namespace and tell him if he has a problem getting it working he can ask for help on the forum.

_However_, dave deploys this vertex ingressroute CRD.
```yaml
apiVersion: contour.heptio.com/v1alpha1
kind: IngressRoute
metadata:
  name: webapp 
  namespace: customer-dave
spec:
  routes:
  - match: /mail
    delegate:
      name: gmail
      namespace: gmail
```
And all of a sudden, `http://dave.cheney.net/mail` is serving up the full gmail application, sans TLS.

This is a large worked example, but the problem boils down to "vertices have no way to list who they are delegated to".
Without a way to deny or permit delegations, anyone who knows the name of the object (which I don't believe is a secret, and even if it was, is security via obscurity) can expose those routes on a virtualhost they own.

### Mitigations

The best way I've thought to mitigate this so far is the vertex ingress routes would have to list the name of either

- the name/namespace of the incoming delegation
- the virtualhost.fqdn at the root of a delegation chain.

This is unfortunate for two reasons:

- It's more boilerplate: a delegates to b, b has to list a as a cross check.
- It turns a DAG into a linked list (possibly not the right term), but it would be impossible for a web service to be a member of multiple roots, unless of course we made the list of incoming vertices be a list -- which would probably push the solution into using virtualhost.fqdn.

The last thing is that this boilerplate _should be required_ even when not in enforcing mode.
I'm not interested in proposing a design where the security interlock that prevents dave's cheap webhosting from publishing gmail without TLS is considered to be the customer's problem because they did not add an optional key.
Said again, if this is the mitigation we choose to adopt, it has to be mandatory for all users, because we all know how effective optional security features are.

# Metrics

Metrics are essential to any system. Contour will expose a `/metrics` Prometheus endpoint with the following metrics:

- **contour_ingressroute_total (gauge):** Number of IngressRoutes
  - namespace
  - vhost
- **contour_ingressroute_upstream_total (gauge):** Number of Upstreams per IngressRoute
  - namespace
  - name
- **contour_ingressroute_upstream_endpoints_total (gauge):** Number of Endpoints per Upstream
  - namespace
  - name
- **contour_ingressroute_upstream_endpoints_healthy (gauge):** Number of healthy Endpoints per Upstream
  - namespace
  - name
- **contour_ingressroute_upstream_endpoints_change (counter):** Number of changes of Endpoints per Upstream
  - namespace
  - name
- **contour_ingressroute_invalid_total (gauge):**  Number of Invalid Routes (Routes who have no root delegating to them)
  - namespace
- **upstream-services**: Number of upstream services, labeled by cluster and tenant/namespace
  - cluster
  - tenant/namespace
- **replicated-services:** Number of services replicated to gimbal cluster labeled by cluster and tenant/namespace
  - cluster
  - tenant/namespace
- **upstream-endpoints:** Number of endpoints - meaning IP:Port - in the upstream labeled by cluster, namespace, and service name
  - cluster
  - tenant/namespace
  - service
- **replicated-endpoints:** Number of endpoints - meaning IP:Port - replicated to Gimbal cluster labeled by cluster, namespace and service name
  - cluster
  - tenant/namespace
  - service

## Envoy Metrics

In addition to the metrics built into Envoy, we'd like to capture metrics per VHost to allows teams to have better visibility into their applications, however, currently they are not exposed. See issue opened upstream with options: [https://github.com/envoyproxy/envoy/issues/3351](https://github.com/envoyproxy/envoy/issues/3351])