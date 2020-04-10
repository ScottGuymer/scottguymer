+++
categories = []
date = 2020-04-09T22:00:00Z
draft = true
keywords = ["tech"]
tags = ["envoy", "security", "authentication"]
title = "Configuring JWT Authentication in Envoy Proxy"

+++
When creating APIs it can be useful to separate out the concern of validating JWT tokens to some downstream service. This has a number of benefits 

* Testing becomes easier as you do not have to create valid JWTs for each API call
* Less configuration needs to be distributed to the API at runtime

You can still pass and make use of JWTs and their payloads within the API but you just offload the task of actually validating that payload to something else.

This sort of functionality, along with things like mutual TLS and other security concerns are starting to become commonplace in service mesh implementations such as Istio.

My needs are slightly different as I am looking to have this in a multi-environment setup primarily using Cloud Foundry and Docker Swarm.

After a little bit of searching looking for a proxy that was able to do this easily, I settled upon Envoy. This was my first time having to get to grips with Envoy so took a little bit of reading and understanding how it all fits together and more than a little bit of trial and error.

## How to configure?

I needed to validate JWT tokens on only certain paths and I wanted to validate based on keys that came from the JWKS URL of an OIDC compatible provider.

The configuration comes in a number of sections

* Configuring the `jwt_authn` HTTP filter
  * The OIDC provider configuration
  * The routes that should be authenticated
* Configuring the cluster to allow the `jwt_authn` filter to query the JWKS keys.
* 