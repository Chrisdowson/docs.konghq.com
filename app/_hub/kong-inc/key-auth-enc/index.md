---
name: Key Authentication - Encrypted
publisher: Kong Inc.
version: 1.0.0

desc: Add key authentication to your Services
description: |
  Add key authentication (also sometimes referred to as an API key) to a Service or a Route. Consumers then add their key either in a query string parameter or a header to authenticate their requests. This plugin is functionally identical to the open source `key-auth` plugin, with the exception that API keys are stored in an encrypted format within the Kong data store.

enterprise: true
alpha: false
type: plugin
categories:
  - authentication

kong_version_compatibility:
    enterprise_edition:
      compatible:
        - 1.5.x
        - 1.3.x

params:
  name: key-auth-enc
  service_id: true
  route_id: true
  consumer_id: false
  protocols: ["http", "https"]

  config:
    - name: key_names
      required: false
      default: "`apikey`"
      description: |
        Describes an array of parameter names where the plugin will look for a key. The client must send the authentication key in one of those key names, and the plugin will try to read the credential from a header or the query string parameter with the same name.<br><br>**Note**: the key names may only contain [a-z], [A-Z], [0-9], [_] and [-].
    - name: key_in_body
      required: false
      default: "`false`"
      description: |
        If enabled, the plugin will read the request body (if said request has one and its MIME type is supported) and try to find the key in it. Supported MIME types are `application/www-form-urlencoded`, `application/json`, and `multipart/form-data`.
    - name: hide_credentials
      required: false
      default: "`false`"
      description: |
        An optional boolean value telling the plugin to show or hide the credential from the upstream service. If `true`, the plugin will strip the credential from the request (i.e. the header or querystring containing the key) before proxying it.
    - name: anonymous
      required: false
      default:
      description: |
        An optional string (consumer uuid) value to use as an "anonymous" consumer if authentication fails. If empty (default), the request will fail with an authentication failure `4xx`. Please note that this value must refer to the Consumer `id` attribute which is internal to Kong, and **not** its `custom_id`.
    - name: run_on_preflight
      required: false
      default: "`true`"
      description: |
        A boolean value that indicates whether the plugin should run (and try to authenticate) on `OPTIONS` preflight requests, if set to `false` then `OPTIONS` requests will always be allowed.
  extra: |
    Note that, according to their respective specifications, HTTP header names are treated as case _insensitive_, while HTTP query string parameter names are treated as case _sensitive_. Kong follows these specifications as designed, meaning that the `key_names` configuration values will be treated differently when searching the request header fields, versus searching the query string. Administrators are advised against defining case-sensitive `key_names` values when expecting the authorization keys to be sent in the request headers.

    <div class="alert alert-warning">
        <center>The option `config.run_on_preflight` is only available from version `0.11.1` and later</center>
    </div>

    Once applied, any user with a valid credential can access the Service.
    To restrict usage to only some of the authenticated users, also add the
    [ACL](/plugins/acl/) plugin (not covered here) and create whitelist or
    blacklist groups of users.

---

## Usage

Prior to creating this plugin, Kong's encryption keyring must be enabled. See the [keyring Getting Started guide](/enterprise/latest/db-encryption#getting-started) for more details.

### Create a Consumer

You need to associate a credential to an existing [Consumer][consumer-object] object.
A Consumer can have many credentials.

{% tabs %}
{% tab With a Database %}
To create a Consumer, execute the following request:

```bash
curl -d "username=user123&custom_id=SOME_CUSTOM_ID" http://kong:8001/consumers/
```

{% endtabs %}

The parameters are as described below:

parameter                       | description
---                             | ---
`username`<br>*semi-optional*   | The username of the consumer. Either this field or `custom_id` must be specified.
`custom_id`<br>*semi-optional*  | A custom identifier used to map the consumer to another database. Either this field or `username` must be specified.

If you are also using the [ACL](/plugins/acl/) plugin and whitelists with this
service, you must add the new consumer to a whitelisted group. See
[ACL: Associating Consumers][acl-associating] for details.

### Create a Key

{% tabs %}
{% tab With a database %}
You can provision new credentials by making the following HTTP request:

```bash
$ curl -X POST http://kong:8001/consumers/{consumer}/key-auth-enc -d ''
HTTP/1.1 201 Created

{
    "consumer": { "id": "876bf719-8f18-4ce5-cc9f-5b5af6c36007" },
    "created_at": 1443371053000,
    "id": "62a7d3b7-b995-49f9-c9c8-bac4d781fb59",
    "key": "62eb165c070a41d5c1b58d9d3d725ca1"
}
```

<div class="alert alert-warning">
  <strong>Note:</strong> It is recommended to let Kong auto-generate the key. Only specify it yourself if you are migrating an existing system to Kong. You must re-use your keys to make the migration to Kong transparent to your Consumers.
</div>

{% endtabs %}
The fields/parameters work as follows:

field/parameter     | description
---                 | ---
`{consumer}`        | The `id` or `username` property of the [Consumer][consumer-object] entity to associate the credentials to.
`key`<br>*optional* | You can optionally set your own unique `key` to authenticate the client. If missing, the plugin will generate one.

### Using the Key

Simply make a request with the key as a querystring parameter:

```bash
$ curl http://kong:8000/{proxy path}?apikey=<some_key>
```

Or in a header:

```bash
$ curl http://kong:8000/{proxy path} \
    -H 'apikey: <some_key>'
```

### Delete a Key

You can delete an API Key by making the following HTTP request:

```bash
$ curl -X DELETE http://kong:8001/consumers/{consumer}/key-auth-enc/{id}
HTTP/1.1 204 No Content
```

* `consumer`: The `id` or `username` property of the [Consumer][consumer-object] entity to associate the credentials to.
* `id`: The `id` attribute of the key credential object.

### Upstream Headers

When a client has been authenticated, the plugin will append some headers to the request before proxying it to the upstream service, so that you can identify the Consumer in your code:

* `X-Consumer-ID`, the ID of the Consumer on Kong
* `X-Consumer-Custom-ID`, the `custom_id` of the Consumer (if set)
* `X-Consumer-Username`, the `username` of the Consumer (if set)
* `X-Credential-Username`, the `username` of the Credential (only if the consumer is not the 'anonymous' consumer)
* `X-Anonymous-Consumer`, will be set to `true` when authentication failed, and the 'anonymous' consumer was set instead.

You can use this information on your side to implement additional logic. You can use the `X-Consumer-ID` value to query the Kong Admin API and retrieve more information about the Consumer.

### Paginate through keys

<div class="alert alert-warning">
  <strong>Note:</strong> This endpoint was introduced in Kong 0.11.2.
</div>

You can paginate through the API keys for all Consumers using the following
request:

```bash
$ curl -X GET http://kong:8001/key-auth-encs

{
   "data":[
      {
         "id":"17ab4e95-9598-424f-a99a-ffa9f413a821",
         "created_at":1507941267000,
         "key":"Qslaip2ruiwcusuSUdhXPv4SORZrfj4L",
         "consumer": { "id": "c0d92ba9-8306-482a-b60d-0cfdd2f0e880" }
      },
      {
         "id":"6cb76501-c970-4e12-97c6-3afbbba3b454",
         "created_at":1507936652000,
         "key":"nCztu5Jrz18YAWmkwOGJkQe9T8lB99l4",
         "consumer": { "id": "c0d92ba9-8306-482a-b60d-0cfdd2f0e880" }
      },
      {
         "id":"b1d87b08-7eb6-4320-8069-efd85a4a8d89",
         "created_at":1507941307000,
         "key":"26WUW1VEsmwT1ORBFsJmLHZLDNAxh09l",
         "consumer": { "id": "3c2c8fc1-7245-4fbb-b48b-e5947e1ce941" }
      }
   ]
   "next":null,
}
```

You can filter the list by consumer by using this other path:

```bash
$ curl -X GET http://kong:8001/consumers/{username or id}/key-auth-enc

{
    "data": [
       {
         "id":"6cb76501-c970-4e12-97c6-3afbbba3b454",
         "created_at":1507936652000,
         "key":"nCztu5Jrz18YAWmkwOGJkQe9T8lB99l4",
         "consumer": { "id": "c0d92ba9-8306-482a-b60d-0cfdd2f0e880" }
       }
    ]
    "next":null,
}
```

`username or id`: The username or id of the consumer whose credentials need to be listed

### Retrieve the Consumer associated with a key

<div class="alert alert-warning">
  <strong>Note:</strong> This endpoint was introduced in Kong 0.11.2.
</div>

It is possible to retrieve a [Consumer][consumer-object] associated with an API
Key using the following request:

```bash
curl -X GET http://kong:8001/key-auth-encs/{key or id}/consumer

{
   "created_at":1507936639000,
   "username":"foo",
   "id":"c0d92ba9-8306-482a-b60d-0cfdd2f0e880"
}
```

* `key or id`: The `id` or `key` property of the API key for which to get the
associated Consumer.

[db-encryption]: /enterprise/latest/db-encryption
[configuration]: /latest/configuration
[consumer-object]: /latest/admin-api/#consumer-object
[acl-associating]: /plugins/acl/#associating-consumers
[faq-authentication]: /about/faq/#how-can-i-add-an-authentication-layer-on-a-microservice/api?
