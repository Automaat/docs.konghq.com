---
title: Sessions in Kong Manager
badge: enterprise
---

When a user logs in to Kong Manager with their credentials, the Sessions Plugin
will create a session cookie. The cookie is used for all subsequent requests and
is valid to authenticate the user. The session has a limited duration and renews
at a configurable interval, which helps prevent an attacker from obtaining and
 using a stale cookie after the session has ended.

The Session configuration is secure by default, which may
[require alteration](#session-security) if using HTTP or different domains for
the Admin API and Kong Manager. Even if an attacker were to obtain a stale
cookie, it would not benefit them since the cookie is encrypted. The encrypted
session data may be stored either in Kong or the cookie itself.

## Configuration the Sessions plugin for Kong Manager

To enable sessions authentication, configure the following:

{% if_version lte:3.1.x %}
```
enforce_rbac = on
admin_gui_auth = <set to desired auth type>
admin_gui_session_conf = {
    "secret":"<SET_SECRET>",
    "cookie_name":"<SET_COOKIE_NAME>",
    "storage":"<SET_STORAGE>",
    "cookie_lifetime":<NUMBER_OF_SECONDS_TO_LIVE>,
    "cookie_secure":<SET_DEPENDING_ON_PROTOCOL>,
    "cookie_samesite":"<SET_DEPENDING_ON_DOMAIN>"
}
```
{% endif_version %}
{% if_version gte:3.2.x lte:3.3.x %}

```
enforce_rbac = on
admin_gui_auth = <set to desired auth type>
admin_gui_session_conf = {
    "secret":"<SET_SECRET>",
    "cookie_name":"<SET_COOKIE_NAME>",
    "storage":"<SET_STORAGE>",
    "rolling_timeout":<NUMBER_OF_SECONDS_UNTIL_RENEWAL>,
    "cookie_secure":<SET_DEPENDING_ON_PROTOCOL>,
    "cookie_same_site":"<SET_DEPENDING_ON_DOMAIN>"
}
```

{% endif_version %}

{% if_version gte:3.4.x %}

```
enforce_rbac = on
admin_gui_auth = <set to desired auth type>
admin_gui_session_conf = {
    "secret":"<SET_SECRET>",
    "cookie_name":"<SET_COOKIE_NAME>",
    "storage":"<SET_STORAGE>",
    "rolling_timeout":<NUMBER_OF_SECONDS_UNTIL_RENEWAL>,
    "cookie_secure":<SET_DEPENDING_ON_PROTOCOL>,
    "cookie_same_site":"<SET_DEPENDING_ON_DOMAIN>",
    "store_metadata": true
}
```

{% endif_version %}

{% if_version lte:3.1.x %}

Attribute | Description
----------|------------
`cookie_name` | A name for the cookie. <br> For example, `"cookie_name":"kong_cookie"`
`secret` | The secret used in keyed HMAC generation. Although the Session plugin's default is a random string, the `secret` _must_ be manually set for use with Kong Manager since it must be the same across all Kong workers/nodes.
`storage` | The location where session data is stored. <br> The default value is `cookie`. It may be more secure if set to `kong`, since access to the database would be required.
`cookie_lifetime` | The duration (in seconds) that the session will remain open. <br> The default value is `3600`.
`cookie_secure` | Applies the Secure directive so that the cookie may be sent to the server only with an encrypted request over the HTTPS protocol. See [Session Security](#session-security) for exceptions. <br> The default value is `true`.
`cookie_samesite`| Determines whether and how a cookie may be sent with cross-site requests. See [Session Security](#session-security) for exceptions. <br> The default value is `strict`.

{% endif_version %}

{% if_version gte:3.2.x lte:3.3.x %}

Attribute | Description
----------|------------
`cookie_name` | A name for the cookie. <br> For example, `"cookie_name":"kong_cookie"`
`secret` | The secret used in keyed HMAC generation. Although the Session plugin's default is a random string, the `secret` _must_ be manually set for use with Kong Manager since it must be the same across all Kong workers/nodes.
`storage` | The location where session data is stored. <br> The default value is `cookie`. It may be more secure if set to `kong`, since access to the database would be required.
`idling_timeout` | The session cookie idle time, in seconds. <br> The default value is 900.
`rolling_timeout` | Specifies, in seconds, how long the session can be used until it needs to be renewed. <br> The default value is 3600.
`cookie_secure` | Applies the Secure directive so that the cookie may be sent to the server only with an encrypted request over the HTTPS protocol. See [Session Security](#session-security) for exceptions. <br> The default value is `true`.
`cookie_same_site`| Determines whether and how a cookie may be sent with cross-site requests. See [Session Security](#session-security) for exceptions. <br> The default value is `strict`.

{% endif_version %}

{% if_version gte:3.4.x %}

Attribute | Description
----------|------------
`cookie_name` | A name for the cookie. <br> For example, `"cookie_name":"kong_cookie"`
`secret` | The secret used in keyed HMAC generation. Although the Session plugin's default is a random string, the `secret` _must_ be manually set for use with Kong Manager since it must be the same across all Kong workers/nodes.
`storage` | The location where session data is stored. <br> The default value is `cookie`. It may be more secure if set to `kong`, since access to the database would be required.
`cookie_lifetime` | The duration (in seconds) that the session will remain open. <br> The default value is `3600`.
`cookie_secure` | Applies the Secure directive so that the cookie may be sent to the server only with an encrypted request over the HTTPS protocol. See [Session Security](#session-security) for exceptions. <br> The default value is `true`.
`cookie_same_site`| Determines whether and how a cookie may be sent with cross-site requests. See [Session Security](#session-security) for exceptions. <br> The default value is `strict`.
`hash_subject`| Whether to hash or not the subject when `store_metadata` is enabled. The default value is `false`.
`store_metadata`| Whether to also store metadata of sessions, such as collecting data of sessions for a specific audience belonging to a specific subject. The default value is `false`. Upon enabling this option, please also set `storage` to `kong`.

{% endif_version %}

{:.important}
> **Important:** The following properties must **not** be altered from default for use with Kong Manager:
* `logout_methods`
* `logout_query_arg`
* `logout_post_arg`

For detailed descriptions of each configuration property, learn more in the
[Session plugin documentation](/hub/kong-inc/session).

{% if_version gte:3.4.x %}

## Multiple session management

When you have multiple sessions running, sessions are isolated by default. You can choose to keep sessions valid or invalidate them on password update:

* **Keep current sessions valid on password update**: Set  `store_metadata` to `false` or  set `storage` to `"cookie". In these cases, the session metadata is not stored on the server side and the session cookie is kept on the client side, so current sessions remain valid.
* **Invalidate current sessions on password update**: Set `storage` to `"kong"` and set `store_metadata` to `true`. In this case, session metadata will be stored on the server side, and all associated sessions will be invalidated upon password updates.

{% endif_version %}

## Session security

The Session configuration is secure by default, so the cookie uses the
[Secure, HttpOnly](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#Secure_and_HttpOnly_cookies),
and [SameSite](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#SameSite_cookies)
directives.

The following properties must be altered depending on the protocol and domains in use:
* If using HTTP instead of HTTPS: `"cookie_secure": false`
{% if_version lte:3.1.x -%}
* If using different domains for the Admin API and Kong Manager: `"cookie_samesite": "off"`
{% endif_version -%}
{% if_version gte:3.2.x -%}
* If using different domains for the Admin API and Kong Manager: `"cookie_same_site": "Lax"`
{% endif_version %}

{:.important}
> **Important:** Sessions are not invalidated when a user logs out if `"storage": "cookie"` (the default) is used. In that case, the cookie is deleted client-side. Only when session data is stored server-side with `"storage": "kong"` set is the session actively invalidated.


## Example configurations

If using HTTPS and hosting Kong Manager and the Admin API from the same domain,
the following configuration could be used for Basic Auth:

```
enforce_rbac = on
admin_gui_auth = basic-auth
admin_gui_session_conf = {
    "cookie_name":"$4m04$",
    "secret":"change-this-secret",
    "storage":"kong"
}
```

In testing, if using HTTP, the following configuration could be used instead:

```
enforce_rbac = on
admin_gui_auth = basic-auth
admin_gui_session_conf = {
    "cookie_name":"04tm34l",
    "secret":"change-this-secret",
    "storage":"kong",
    "cookie_secure":false
}
```

{% if_version gte:3.4.x %}
If you want to invalidate all sessions when the password is changed, the following configuration could be used with the Basic Auth plugin:

```
enforce_rbac = on
admin_gui_auth = basic-auth
admin_gui_session_conf = {
    "cookie_name":"kong_cookie",
    "secret":"change-this-secret",
    "storage":"kong",
    "cookie_secure":false,
    "store_metadata":true
}
```

If you need to avoid storing session subjects as clear text due to PII concerns with `store_metadata` enabled, the following configuration can be used instead:

```
enforce_rbac = on
admin_gui_auth = basic-auth
admin_gui_session_conf = {
    "cookie_name":"kong_cookie",
    "secret":"change-this-secret",
    "storage":"kong",
    "cookie_secure":false,
    "hash_subject":true,
    "store_metadata":true
}
```

{% endif_version %}
