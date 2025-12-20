
# Amend Proxy Server

A little proxy server to patch HTTP requests before they reach their destination. Expected use case is for JSON payload modifications, though can be used for other purposes.

Environment variables are used to define routing and transformation rules.

## Rule syntax

Environment variable names should begin with `AMEND_` and follow the syntax:

```bash
AMEND_<name>="IN_PORT OUT_HOST PATH_REGEX JS_EXPRESSION"
```

For example, invoking via the shell might look something like:

```bash
AMEND_0_DEFAULTROUTE="8080 http://example.com:80 .*" \
AMEND_1_FOOBAR="8080 http://example.com:80 foo/bar _.foo = 'bar'; console.log(JSON.stringify(_));" \
node ./amend.js
curl -X POST http://localhost:8080/foo/bar \
    -H "Content-Type: application/json" \
    -d '{"name": "John", "age": 30 }'
```

### Components

- `IN_PORT` Specifies the port number the proxy server will listen to for incoming requests.
- `OUT_HOST` Hostname or IP address of the upstream server where requests should be forwarded.
- `PATH_REGEX` Regular expression to match against incoming request URLs to determine if the rule applies.
- `JS_EXPRESSION` JavaScript code, executed in the context of each matching request, allowing custom modifications to requests before forwarding.

### Evaluation context for `JS_EXPRESSION`

- `$` represents the parsed JSON body of the request, if available. If the
  body is not valid JSON or is absent, `$` is undefined.
- `_` a modifiable variable that initially holds a deep copy of `$`, if available.
  Modifications to `_` will be used as the request body when forwarding the
  request to the upstream server.

There are also analogues for the header map (`_headers` and `$headers`), HTTP method (`_method` and `$method`) and URL (`_url` and `$url`). Refer to the source for details.

## Matching and precedence

Matching rules for a request are evaluated in order, meaning that rule occurring later will override the effects of earlier rules.

Rule order is defined by a lexicographic sort of the `AMEND_` environment variable keys, with a special handling for numbers such that e.g. `1 < 10`.

If no rule matches an incoming request, an error is logged to the console and the server responds with an HTTP 500.

