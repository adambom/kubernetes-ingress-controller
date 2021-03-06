# Custom Resource Definitions

The Ingress Controller can configure Kong specific features
using several [Custom Resource Definitions(CRDs)][k8s-crd].

Following CRDs enables users to declaratively configure all aspects of Kong:

- [**KongPlugin**](#kongplugin): This resource corresponds to
  the [Plugin][kong-plugin] entity in Kong.
- [**KongIngress**](#kongingress): This resource provides fine-grained control
  over all aspects of proxy behaviour like routing, load-balancing,
  and health checking. It serves as an "extension" to the Ingress resources
  in Kubernetes.
- [**KongConsumer**](#kongconsumer):
  This resource maps to the [Consumer][kong-consumer] entity in Kong.
- [**KongCredential**](#kongcredential): This resource maps to
  a credential (key-auth, basic-auth, jwt, hmac-auth) that is associated with
  a specific KongConsumer.

## KongPlugin

This resource provides an API to configure plugins inside Kong using
Kubernetes-styled APIs.

Please see the [concept](../concepts/custom-resources.md#KongPlugin)
document for how the resource should be used.

The following snippet shows the properties available:

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: <object name>
  namespace: <object namespace>
  labels:
    global: "true"   # optional, if set, then the plugin will be executed
                     # for every request that Kong proxies
                     # please note the quotes around true
disabled: <boolean>  # optionally disable the plugin in Kong
config:              # configuration for the plugin
    key: value
plugin: <name-of-plugin> # like key-auth, rate-limiting etc
```

- `config` contains a list of `key` and `value`
  required to configure the plugin.
  All configuration values specific to the type of plugin go in here.
  Please read the documentation of the plugin being configured to set values
  in here.
- `plugin` field determines the name of the plugin in Kong.
  This field was introduced in Kong Ingress Controller 0.2.0.
- Setting a label `global` to `"true"` will result in the plugin being
  applied globally in Kong, meaning it will be executed for every
  request that is proxied via Kong.

**Please note:** validation of the configuration fields is left to the user.
Setting invalid fields will result in errors in the Ingress Controller.
This behavior is set to improve in the future.

The plugins can be associated with Ingress
or Service object in Kubernetes using `plugins.konghq.com` annotation.

*Example:*

Given the following plugin:

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: request-id
config:
  header_name: my-request-id
plugin: correlation-id
```

It can be applied to a service by annotating like:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  labels:
     app: myapp-service
  annotations:
     plugins.konghq.com: request-id
spec:
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: myapp-service
  selector:
    app: myapp-service
```

It can be applied to a specific ingress (route or routes):

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo-example-com
  annotations:
    plugins.konghq.com: request-id
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /bar
        backend:
          serviceName: echo
          servicePort: 80
```

A plugin can also be applied to a specific KongConsumer by adding
`plugins.konghq.com` annotation to the KongConsumer resource.

Please follow the
[Using the KongPlugin resource](../guides/using-kongplugin-resource.md)
guide for details on how to use this resource.

## KongIngress

Ingress resource spec in Kubernetes can define routing policies
based on HTTP Host header and paths.  
While this is sufficient in most cases,
sometimes, users may want more control over routing at the Ingress level.
`KongIngress` serves as an "extension" to Ingress resource.
It is not meant as a replacement to the
`Ingress` resource in Kubernetes.

Please read the [concept](../concepts/custom-resources.md#kongingress)
document for why this resource exists and how it relates to the existing
Ingress resource.

Using `KongIngress`, all properties of [Upstream][kong-upstream],
[Service][kong-service] and
[Route][kong-route] entities in Kong related to an Ingress resource
can be modified.

Once a `KongIngress` resource is created, it needs to be associated with
an Ingress or Service resource using the following annotation:

```yaml
configuration.konghq.com: kong-ingress-resource-name
```

Specifically,

- To override any properties related to health-checking, load-balancing,
  or details specific to a service, add the annotation to the Kubernetes
  Service that is being exposed via the Ingress API.
- To override routing configuration (like protocol or method based routing),
  add the annotation to the Ingress resource.

Please follow the
[Using the KongIngress resource](../guides/using-kongingress-resource.md)
guide for details on how to use this resource.

For reference, the following is a complete spec for KongIngress:

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongIngress
metadata:
  name: configuration-demo
upstream:
  hash_on: none
  hash_fallback: none
  healthchecks:
    active:
      concurrency: 10
      healthy:
        http_statuses:
        - 200
        - 302
        interval: 0
        successes: 0
      http_path: "/"
      timeout: 1
      unhealthy:
        http_failures: 0
        http_statuses:
        - 429
        interval: 0
        tcp_failures: 0
        timeouts: 0
    passive:
      healthy:
        http_statuses:
        - 200
        successes: 0
      unhealthy:
        http_failures: 0
        http_statuses:
        - 429
        - 503
        tcp_failures: 0
        timeouts: 0
    slots: 10
proxy:
  protocol: http
  path: /
  connect_timeout: 10000
  retries: 10
  read_timeout: 10000
  write_timeout: 10000
route:
  methods:
  - POST
  - GET
  regex_priority: 0
  strip_path: false
  preserve_host: true
  protocols:
  - http
  - https
```

## KongConsumer

This custom resource configures a consumer in Kong:

The following snippet shows the field available in the resource:

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: <object name>
  namespace: <object namespace>
username: <user name>
custom_id: <custom ID>
```

An example:

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: consumer-team-x
username: team-X
```

When this resource is created, a corresponding consumer entity will be
created in Kong.

## KongCredential

This custom resource can be used to configure a consumer specific
entities in Kong.
The resource reference the KongConsumer resource via the `consumerRef` key.

The validation of the config object is left up to the user.

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongCredential
metadata:
  name: credential-team-x
consumerRef: consumer-team-x
type: key-auth
config:
  key: 62eb165c070a41d5c1b58d9d3d725ca1
```

The following credential types can be provisioned using the KongCredential
resource:

- `key-auth` for [Key authentication](https://docs.konghq.com/plugins/key-authentication/)
- `basic-auth` for [Basic authenticaiton](https://docs.konghq.com/plugins/basic-authentication/)
- `hmac-auth` for [HMAC authentication](http://docs.konghq.com/plugins/hmac-authentication/)
- `jwt` for [JWT based authentication](http://docs.konghq.com/plugins/jwt/)
- `oauth2` for [Oauth2 Client credentials](https://docs.konghq.com/hub/kong-inc/oauth2/)
- `acls` for [ACL group associations](https://docs.konghq.com/hub/kong-inc/acl/)

Please ensure that all fields related to the credential in Kong
are present in the definition of KongCredential's `config` section.

Please refer to the
[using the Kong Consumer and Credential resource](../guides/using-consumer-credential-resource.md)
guide for details on how to use this resource.

[k8s-crd]: https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/
[kong-consumer]: https://getkong.org/docs/latest/admin-api/#consumer-object
[kong-plugin]: https://getkong.org/docs/latest/admin-api/#plugin-object
[kong-upstream]: https://getkong.org/docs/latest/admin-api/#upstream-objects
[kong-service]: https://getkong.org/docs/latest/admin-api/#service-object
[kong-route]: https://getkong.org/docs/latest/admin-api/#route-object
