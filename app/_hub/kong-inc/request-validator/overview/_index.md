---
nav_title: Overview
---

Validate requests before they reach their upstream service. This plugin supports validating
the schema of the body and the parameters of the request using either Kong's own
schema validator (body only) or a JSON Schema Draft 4 compliant validator.

## Content-Type validation

The request Content-Type header is validated against the plugin's [`allowed_content_types`](/hub/kong-inc/request-validator/configuration/#config-allowed_content_types) setting. If the request Content-Type is not listed, the request will be rejected with an HTTP/400 error: `{"message":"specified Content-Type is not allowed"}`

The parameter is strictly validated, which means a request with a parameter (for example, `application/json; charset=UTF-8`) is NOT considered valid for one without the same parameter (for example, `application/json`). The type, subtype, parameter names, and the value of the charset parameter are not case sensitive based on the RFC explanation.

{:.important}
> When setting this configuration, the `Content-Type` header only gets validated when `body_schema` is configured.

## Body validation

{% if_version lte:3.5.x %}
Request body validation is only performed for requests with `application/json` Content-Type.
{% endif_version %}
{% if_version gte:3.6.x %}
Request body validation is only performed for requests with `application/json` Content-Type or when `+json` is suffixed in the request Content-Type’s subtype (for example, `application/merge-patch+json`).
{% endif_version %}

For requests with any other allowed Content-Type, body validation is skipped. In that case, the request is proxied to  the upstream without validating the body.

Either Kong's own schema validator (`config.version=kong`) or a JSON Schema Draft 4 compliant validator (`config.version=draft4`) can be used to validate the request body.

## Parameter validation

Only the JSON Schema Draft 4 compliant validator is supported for parameter validation.

{:.important}
> Even if `config.version` is set to `kong`, the parameter validation will still use the JSON Schema Draft 4 compliant validator.

## Examples

### Overview

By applying the plugin to a service, all requests to that service will be validated
before being proxied.

{% navtabs %}
{% navtab With a database %}

Use a request like this:

``` bash
curl -i -X POST http://localhost:8001/services/{service}/plugins \
  --data "name=request-validator" \
  --data "config.version=kong" \
  --data 'config.body_schema=[{\"name\":{\"type\": \"string\", \"required\": true}}]'
```
{% endnavtab %}

{% navtab Without a database %}

Add the following entry to the `plugins:` section in the declarative configuration file:

``` yaml
plugins:
- name: request-validator
  service: {service}
  config:
    version: kong
    body_schema:
      name:
        type: string
        required: true
```
{% endnavtab %}
{% endnavtabs %}

In this example, the request body data would have to be valid JSON and
conform to the schema specified in `body_schema` - i.e., it would be required
to contain a `name` field only, which needs to be a string.

In case the validation fails, a `400 Bad Request` will be returned as the response.

### Schema definition

*For using the JSON Schema Draft 4-compliant validator, see the [JSON Schema website](
https://json-schema.org/) for details on the format and examples. The rest of
this paragraph will explain the Kong schema.*

The `config.body_schema` field expects a JSON array with the definition of each
field expected to be in the request body; for example:

Each field definition contains the following attributes:

| Attribute | Required | Description |
| --- | --- | --- |
| `type` | yes | The expected type for the field |
| `required` | no | Whether the field is required |

Additionally, specific types may have their own required fields:

Map:

| Attribute | Required | Description |
| --- | --- | --- |
| `keys` | yes | The schema for the map keys |
| `values` | yes | The schema for the map values |

Example string-boolean map:

```json
{
  "type": "map",
  "keys": {
    "type": "string"
  },
  "values": {
    "type": "boolean"
  }
}
```

Array:

| Attribute | Required | Description |
| --- | --- | --- |
| `elements` | yes | The array's elements schema |

Example integer array schema:

```json
{
  "type": "array",
  "elements": {
    "type": "integer"
  }
}
```

Record:

| Attribute | Required | Description |
| --- | --- | --- |
| `fields` | yes | The record schema |

Example record schema:

```json
{
  "type": "record",
  "fields": [
    {
      "street": {
        "type": "string",
      }
    },
    {
      "zipcode": {
        "type": "string",
      }
    }
  ]
}
```


The `type` field assumes one the following values:

- `string`
- `number`
- `integer`
- `boolean`
- `map`
- `array`
- `record`

Each field specification may also contain validators, which perform specific
validations:

| Validator | Applies to | Description |
| --- | --- | --- |
| `between` | Integers | Whether the value is between two integers. Specified as an array; for example, `{1, 10}` |
| `len_eq` | Arrays, Maps, Strings | Whether an array's length is a given value |
| `len_min` | Arrays, Maps, Strings | Whether an array's length is at least a given value |
| `len_max` | Arrays, Maps, strings | Whether an array's length is at most a given value |
| `match` | Strings | True if the value matches a given Lua pattern ** |
| `not_match` | String | True if the value doesn't match a given Lua pattern ** |
| `match_all` | Arrays | True if all strings in the array match the specified Lua pattern ** |
| `match_none` | Arrays | True if none of the strings in the array match the specified Lua pattern ** |
| `match_any` | Arrays | True if any one of the strings in the array matches the specified Lua pattern ** |
| `starts_with` | Strings | True if the string value starts with the specified substring |
| `one_of` | Strings, Numbers, Integers | True if the string field value matches one of the specified values |
| `timestamp` | Integers | True if the field value is a valid timestamp |
| `uuid`| Strings | True if the string is a valid UUID |

**Note**: To learn more, see [Lua patterns][lua-patterns].

#### Semantic validation for the JSON Schema `format` attribute

{:.important}
> This feature is only supported in JSON Schema Draft 4.

Structural validation alone may be insufficient to validate that an instance
meets all the requirements of an application. The `format` keyword is defined
to allow interoperable semantic validation for a fixed subset of values that
are accurately described by authoritative resources, be they RFCs or other
external specifications. The following attributes are available:

| Attribute | Description |
| --- | --- |
| `date` | defined by [RFC 3339], sections [5.6] and further validated by [5.7] |
| `date-time` | defined by [RFC 3339], sections [5.6] |
| `time` | defined by [RFC 3339], sections [5.6] and further validated by [5.7] |

**Note**: The value of the `format` attribute must be a string.

Example `date` schema:

```json
{
  "type": "string",
  "format": "date"
}
```

### JSON Schema example

```json
[
  {
    "name": {
      "type": "string",
      "required": true
    }
  },
  {
    "age": {
      "type": "integer",
      "required": true
    }
  },
  {
    "address": {
      "type": "record",
      "required": true,
      "fields": [
        {
          "street": {
            "type": "string",
            "required": true
          }
        },
        {
          "zipcode": {
            "type": "string",
            "required": true
          }
        }
      ]
    }
  },
  {
    "born": {
      "type": "string",
      "format": "date-time",
      "required": true
    }
  }
]
```

Such a schema would validate the following request body:

```json
{
  "name": "Gruce The Great",
  "age": 4,
  "address": {
    "street": "251 Post St.",
    "zipcode": "94108"
  },
  "born": "2009-07-20T08:30:37.012Z"
}

```

### Parameter schema definition

You can set up definitions for each parameter based on the OpenAPI Specification and
the plugin will validate each parameter against it. For more information, see the
[OpenAPI specification](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.2.md#parameter-object)
or the [OpenAPI examples](https://swagger.io/docs/specification/serialization/).

#### Fixed fields

|Field Name | Type | Description|
| --- | --- | --- |
|name | `string` | **REQUIRED**. The name of the parameter. Parameter names are *case sensitive*, and corresponds to the parameter name used by the `in` property. If `in` is `"path"`, the `name` field MUST correspond to the [named capture group](/gateway/latest/how-kong-works/routing-traffic/#capturing-groups) from the configured `route`.|
|in | `string` | **REQUIRED**. The location of the parameter. Possible values are `query`, `header`, or `path`.|
|required | `boolean` | **REQUIRED** Determines whether this parameter is mandatory.|
|style | `string` | **REQUIRED** when schema and explode are set<br> Describes how the parameter value will be deserialized depending on the type of the parameter value.|
|schema | `string` | **REQUIRED** when style and explode are set<br> The schema defining the type used for the parameter. It is validated using `draft4` for JSON Schema draft 4 compliant validator. In addition to being a valid JSON Schema, the parameter schema MUST have a top-level `type` property to enable proper deserialization before validating. |
|explode | `boolean` | **REQUIRED** when schema and style are set<br> When this is true, parameter values of type `array` or `object` generate separate parameters for each value of the array or key-value pair of the map.  For other types of parameters this property has no effect.|

#### Examples

In this example, use the plugin to validate a request's path parameter.

1.  Add a service to Kong:

    ```sh
    curl -i -X POST http://localhost:8001/services \
      --data name=httpbin \
      --data url=https://httpbin.konghq.com
    ```

2.  Add a route with [named capture group](/gateway/latest/how-kong-works/routing-traffic/#capturing-groups):

    ```sh
    curl -i -X POST http://localhost:8001/services/httpbin/routes \
      --data paths="~/status/(?<status_code>\d%+)" \
      --data strip_path=false
    ```

3. Enable `request-validator` plugin to validate body and parameter:

    ```sh
    curl -i -X POST http://localhost:8001/services/httpbin/plugins \
      --header "Content-Type: application/json" \
      --data @parameter_schema.json
    ```

    Content of the file `parameter_schema.json`:

    ```json
    {
      "name": "request-validator",
      "config": {
        "body_schema": "{\"properties\":{\"name\":{\"type\":\"string\"}},\"required\":[\"name\"]}",
        "version": "draft4",
        "parameter_schema": [
          {
            "name": "status_code",
            "in": "path",
            "required": true,
            "schema": "{\"type\": \"number\"}",
            "style": "simple",
            "explode": false
          }
        ]
      }
    }
    ```

4. In these step examples, validation ensures that `status_code` is a number and the body contains a parameter called `name`.

   For example, if you try to send a request to an existing route with the path `/status/abc`:

    ```sh
    curl -i -X POST \
    --url http://localhost:8000/status/abc \
    --header 'Content-Type: application/json' \
    --data '{ "name": "foo" }'
    ```

    The proxy request is blocked because the path has a non-numerical status code, which doesn't conform to the schema:
    ```
    HTTP/1.1 400 Bad Request
    ...

    {"message":"request param doesn't conform to schema"}
    ```

    A proxy request with a numeric status code is allowed:

    ```sh
    curl -i -X POST \
    --url http://localhost:8000/status/123 \
    --header 'Content-Type: application/json' \
    --data '{ "name": "foo" }'
    ```

    Response:
    ```
    HTTP/1.1 200 OK
    ```

### Further references

The Kong schema validation format is based on the plugin schemas.
For more information, see the Kong plugin docs on
[storing custom entities](/gateway/latest/plugin-development/custom-entities/#defining-a-schema).

---

[schema-docs]: /1.0.x/plugin-development/custom-entities/#defining-a-schema
[lua-patterns]: https://www.lua.org/pil/20.2.html
[RFC 3339]: https://tools.ietf.org/html/rfc3339
[5.6]: https://tools.ietf.org/html/rfc3339#section-5.6
[5.7]: https://tools.ietf.org/html/rfc3339#section-5.7
