+++
date = 2020-04-09T22:00:00Z
keywords = ["tech"]
tags = ["envoy", "security", "authentication"]
title = "Configuring JWT Authentication in Envoy Proxy"
+++

When creating APIs it can be useful to separate out the concern of validating JWT tokens to some downstream service. This has a number of benefits

- Testing becomes easier as you do not have to create valid JWTs for each API call
- Less configuration needs to be distributed to the API at runtime

You can still pass and make use of JWTs and their payloads within the API but you just offload the task of actually validating that payload to something else.

This sort of functionality, along with things like mutual TLS and other security concerns are starting to become commonplace in service mesh implementations such as Istio.

My needs are slightly different as I am looking to have this in a multi-environment setup primarily using Cloud Foundry and Docker Swarm.

After a little bit of searching looking for a proxy that was able to do this easily, I settled upon Envoy. This was my first time having to get to grips with Envoy so took a little bit of reading and understanding how it all fits together and more than a little bit of trial and error.

## How to configure?

I needed to validate JWT tokens on only certain paths and I wanted to validate based on keys that came from the JWKS URL of an OIDC compatible provider.

The configuration comes in a number of sections

- Configuring the `jwt_authn` HTTP filter
  - The OIDC provider configuration
  - The routes that should be authenticated
- Configuring the cluster to allow the `jwt_authn` filter to query the JWKS keys.

### The HTTP filter

Inside your `config.yml` you need to add a new HTTP filter. This needs to be above the `envoy.router` filter and anything else that you might have in there that you want the security to run before.

```yaml
http_filters:
  - name: envoy.filters.http.jwt_authn
    config:
      providers:
        oidc_provider:
          issuer: my_issuer_name
          audiences:
            - audience_to_validate
          forward: 'true'
          remote_jwks:
            http_uri:
              uri: https://my.provider/jwks_url
              cluster: identity_provider
              timeout: 5s
            cache_duration:
              seconds: 300
      rules:
        - match: { prefix: /api }
          requires:
            provider_name: oidc_provider
      bypass_cors_preflight: 'true'
```

To explain this config

- `providers:` section describes the (1 or more) providers that can be used to validated tokens passed on requests that go through this HTTP filter.
  - `issuer:` is the exact value of the `iss` property in the tokens to be validated. This is usually a URL
  - `audiences:` a list of valid audiences that can be in the `aud` value in the JWT
  - `forward:` true here means that the `Authorization` header will be forwarded to the upstream service. This is useful for later use.
  - `remote_jwks:` this is used to configure Envoy to retrieve the signing keys directly from the IdP and cache them locally. There are other possibilities on how to configure this for instance with a static set of key values. Here you need to provide the URL to query the keys from and the cluster configuration (configured later) that will be used to query the URL. We also configure the cache duration here to 5 mins which should be more than enough.
- `rules:` represents the set of rules used to decide which requests will need to be validated and which can be simply passed on. In my case, I wanted to validate everything on `/api` and I want to validate using the provider we just configured above.
- `bypass_cors_preflight` This is used to tell envoy to bypass validation on any CORS preflight requests issued by the browser

### Cluster Configuration

In my case, I wanted to automatically retrieve the latest signing keys from the JWKS URL of my OIDC IdP. To enable this you need to create a cluster specifically for the IdP domain and port.

```yaml
clusters:
  - name: identity_provider
    connect_timeout: 0.25s
    type: STRICT_DNS
    load_assignment:
      cluster_name: identity_provider
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: my.provider
                    port_value: 443
    transport_socket:
      name: envoy.transport_sockets.tls
```

This cluster has the same name as the cluster identified in the `remote_jwks` configuration above. Here we set the endpoint of this cluster to be the domain of the URL of the IdP configured above. We also enable SSL with the `transport_socket` config.
