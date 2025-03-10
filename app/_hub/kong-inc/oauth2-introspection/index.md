---

name: OAuth 2.0 Introspection
publisher: Kong Inc.
version: 1.3-x

desc: Integrate Kong with a third-party OAuth 2.0 Authorization Server
description: |
  Validate access tokens sent by developers using a third-party OAuth 2.0
  Authorization Server, by leveraging its Introspection Endpoint
  ([RFC 7662](https://tools.ietf.org/html/rfc7662)). This plugin assumes that
  the Consumer already has an access token that will be validated against a
  third-party OAuth 2.0 server.

enterprise: true
type: plugin
categories:
  - authentication

kong_version_compatibility:
    community_edition:
      compatible:
    enterprise_edition:
      compatible:
        - 1.5.x
        - 1.3-x
        - 0.36-x
        - 0.35-x
        - 0.34-x

params:
  name: oauth2-introspection
  api_id: true
  service_id: true
  route_id: true
  consumer_id: true
  config:
    - name: introspection_url
      required:
      default:
      value_in_examples:
      description: |
        The full URL to the third-party introspection endpoint
    - name: authorization_value
      required:
      default:
      value_in_examples:
      description: |
        The value to append to the Authorization header when requesting the introspection endpoint
    - name: token_type_hint
      required: false
      default:
      value_in_examples:
      description: |
        The token_type_hint value to associate to introspection requests
    - name: ttl
      required: false
      default: 60
      value_in_examples:
      description: |
        The TTL in seconds for the introspection response - set to 0 to disable the expiration
    - name: hide_credentials
      required: false
      default:
      value_in_examples:
      description: |
        An optional boolean value telling the plugin to hide the credential to the upstream API server. It will be removed by Kong before proxying the request.
    - name: timeout
      required: false
      default: 10000
      value_in_examples:
      description: |
        An optional timeout in milliseconds when sending data to the upstream server
    - name: keepalive
      required: false
      default: 60000
      value_in_examples:
      description: |
        An optional value in milliseconds that defines for how long an idle connection will live before being closed
    - name: anonymous
      required: false
      default:
      value_in_examples:
      description: |
        An optional string (consumer uuid) value to use as an "anonymous" consumer if authentication fails. If empty (default), the request will fail with an authentication failure 4xx.
    - name: run_on_preflight
      required: false
      default: true
      value_in_examples:
      description: |
        A boolean value that indicates whether the plugin should run (and try to authenticate) on `OPTIONS` preflight requests. If set to `false` then `OPTIONS` requests will always be allowed.
    - name: consumer_by
      required: false
      default: username
      value_in_examples: username
      description: |
        A string indicating whether to associate oauth2 `username` or `client_id`
        with the consumer's username. Oauth2 `username` is mapped to a consumer's
        `username` field, while an oauth2 `client_id` maps to a consumer's
        `custom_id`
    - name: introspect_request
      required: false
      default: false
      value_in_examples:
      description: |
        A boolean indicating whether to forward information about the current
        downstream request to the introspect endpoint. If true, headers
        `X-Request-Path` and `X-Request-Http-Method` will be inserted in the
        introspect request
    - name: custom_introspection_headers
      required: false
      default:
      value_in_examples:
      description: |
        A list of custom headers to be added in the introspection request
    - name: custom_claims_forward
      required: false
      default:
      value_in_examples:
      description: |
        A list of custom claims to be forwarded from the introspection response
        to the upstream request. Claims are forwarded in headers with prefix
        `X-Credential-{claim-name}`
---

### Flow

![OAuth2 Introspection Flow](/assets/images/docs/oauth2/oauth2-introspection.png)

### Associate the response to a Consumer

To associate the introspection response resolution to a Kong Consumer, you will have to provision a Kong Consumer with the same `username` returned by the Introspection Endpoint response.

### Upstream Headers

When a client has been authenticated, the plugin will append some headers to the request before proxying it to the upstream API/Microservice, so that you can identify the consumer in your code:

- `X-Consumer-ID`, the ID of the Consumer on Kong (if matched)
- `X-Consumer-Custom-ID`, the `custom_id` of the Consumer (if matched and if existing)
- `X-Consumer-Username`, the `username of` the Consumer (if matched and if existing)
- `X-Anonymous-Consumer`, will be set to true when authentication failed, and the 'anonymous' consumer was set instead.
- `X-Credential-Scope`, as returned by the Introspection response (if any)
- `X-Credential-Client-ID`, as returned by the Introspection response (if any)
- `X-Credential-Username`, as returned by the Introspection response (if any)
- `X-Credential-Token-Type`, as returned by the Introspection response (if any)
- `X-Credential-Exp`, as returned by the Introspection response (if any)
- `X-Credential-Iat`, as returned by the Introspection response (if any)
- `X-Credential-Nbf`, as returned by the Introspection response (if any)
- `X-Credential-Sub`, as returned by the Introspection response (if any)
- `X-Credential-Aud`, as returned by the Introspection response (if any)
- `X-Credential-Iss`, as returned by the Introspection response (if any)
- `X-Credential-Jti`, as returned by the Introspection response (if any)

Additionally, any claims specified in `config.custom_claims_forward` will also be forwarded with the `X-Credential-` prefix.

Note: Aforementioned `X-Credential-*` headers are not set when authentication failed, and the 'anonymous' consumer was set instead.
