---
name: openapi-docs
description: Write, generate, review, and fix API endpoint documentation as OpenAPI 3.1 specs (JSON or YAML), including request/response examples, schemas, auth, and error shapes. Use this skill whenever the user mentions API docs, endpoint documentation, OpenAPI, Swagger, an openapi.json or openapi.yaml file, "document my API", "spec for my routes", or wants example requests and responses for HTTP endpoints. Also use it when the user pastes route handlers, curl commands, sample JSON payloads, or server code (C, C++, Node, Python, Go, etc.) and wants them turned into documentation, even if they never say "OpenAPI".
---

# OpenAPI Docs

Turn an API (source code, curl calls, sample payloads, or a rough description) into an accurate, valid OpenAPI 3.1 document that renders cleanly in Swagger UI, Redoc, Scalar, Postman, and Bruno, and that code generators can consume.

## Why this matters

API docs are only useful if they are correct and machine-readable. A spec that drifts from the real behavior is worse than none, so accuracy beats completeness: document what the API actually does, and flag what you had to guess.

## Workflow

### 1. Gather the facts
Collect, from whatever the user provided:
- Base URL(s) and environments (local, staging, prod)
- Every route: method + path
- Path, query, and header parameters (type, required, defaults, constraints)
- Request bodies and their content types
- Response status codes and bodies, including errors
- Auth scheme (none, API key, bearer/JWT, OAuth2, session cookie)
- Pagination, filtering, sorting, rate limiting conventions

If the user gave source code, read the handlers to extract these. Do not invent fields that the code does not produce. If something is ambiguous (for example, whether a field is nullable), pick the safest reading, mark it in the output, and list it under "Assumptions" at the end of your reply.

Ask a clarifying question only if you cannot produce a useful draft without the answer. Otherwise draft first and note assumptions.

### 2. Choose format and layout
- **Default to YAML** for hand-edited specs (more readable, supports comments). Use **JSON** when the user says JSON, or when the file will be consumed or generated programmatically.
- **Default to a single file** (`openapi.yaml`). Suggest splitting into multiple files with `$ref` only when the spec exceeds roughly 800 lines.
- Target **OpenAPI 3.1.x**. It aligns fully with JSON Schema 2020-12, so use `type: ["string", "null"]` for nullable fields, not the 3.0-style `nullable: true`.

### 3. Write the spec
Follow the structure and rules below.

### 4. Validate
Run a validator if tools are available:
```bash
npx @redocly/cli lint openapi.yaml
# or
npx @stoplight/spectral-cli lint openapi.yaml
```
If you cannot run tools, re-read the output against the checklist at the bottom before delivering.

### 5. Deliver
- Save the spec as a file (`openapi.yaml` or `openapi.json`) and present it.
- Add a short note: what is covered, assumptions made, and how to preview it (see "Previewing").

## Spec structure

Use this top-level order. It reads naturally and matches what most tools emit.

```yaml
openapi: 3.1.0
info:
  title: <API name>
  version: <semver>
  description: <one or two sentences on what the API is for>
servers:
  - url: http://localhost:8080
    description: Local development
tags:
  - name: tasks
    description: Task management
security:            # global default, omit if the API is public
  - bearerAuth: []
paths:
  ...
components:
  securitySchemes: {}
  parameters: {}
  schemas: {}
  responses: {}
  examples: {}
```

## Rules for writing paths

- **Every operation gets:** `operationId` (unique, camelCase, verb-first like `listTasks`), `summary` (under 60 characters), `tags`, and `responses`. Code generators name their methods from `operationId`, so a missing or sloppy one produces ugly SDKs.
- **Add `description`** only when the summary is not enough: side effects, idempotency, ordering guarantees, or gotchas.
- **Document at least one success response and every error the handler can return.** Do not stop at 200. At minimum consider 400, 401, 403, 404, 409, 422, 429, and 500 where they apply.
- **Path parameters** are always `required: true`.
- **Reuse with `$ref`.** Anything that appears twice (a model, a pagination parameter, an error response) belongs in `components`. Duplicated schemas drift apart.
- **Group with tags** by resource, not by HTTP method.

## Rules for schemas

- Use JSON Schema keywords to carry real constraints: `minLength`, `maxLength`, `minimum`, `maximum`, `pattern`, `enum`, `format` (`date-time`, `email`, `uuid`, `uri`).
- Mark `required` fields explicitly. If it is not in `required`, clients assume it can be absent.
- Give every property a `description` when the name is not self-explanatory. Skip obvious ones like `id`.
- Set `additionalProperties: false` on request bodies when the server rejects unknown fields.
- Separate input and output models when they differ (for example, `TaskCreate` vs `Task`), because read-only fields such as `id` and `created_at` should not appear in request bodies.
- Use `readOnly: true` / `writeOnly: true` for single fields when splitting models is overkill.

## Rules for examples

Examples are the part readers use most, so make them realistic and consistent.

- Put an `example` or `examples` on every request body and every success response. Add examples for common errors too.
- Use believable values (`"Write the docs"`, not `"string"`), and keep the same sample entities across endpoints so the docs tell one coherent story. If `GET /tasks/1` returns `"title": "Write docs"`, the list endpoint should include that same item.
- Every example must validate against its own schema. Check types, required fields, and enum values.
- Use `examples` (plural, named) when showing multiple scenarios, such as `empty list` vs `populated list`.
- Also provide a **curl** snippet per endpoint in the reply or in `x-codeSamples` (see below) when the user wants copy-paste usage.

```yaml
x-codeSamples:
  - lang: Shell
    label: curl
    source: |
      curl -X POST http://localhost:8080/tasks \
        -H "Content-Type: application/json" \
        -d '{"title": "Write docs"}'
```

## Errors: pick one shape and stick to it

Prefer **RFC 9457 Problem Details** (`application/problem+json`) for new APIs. If the API already returns its own error format, document that exactly and do not "improve" it.

```yaml
components:
  schemas:
    Error:
      type: object
      required: [code, message]
      properties:
        code:
          type: string
          description: Stable machine-readable error code
          examples: [TASK_NOT_FOUND]
        message:
          type: string
          description: Human-readable explanation
          examples: [No task with id 42]
  responses:
    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/Error"
```

## Authentication

Declare schemes once in `components.securitySchemes`, then apply them globally or per operation.

```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
    apiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key
```

Public endpoints inside an otherwise protected API override with `security: []`.

## Pagination and lists

Document whichever pattern the API really uses (offset/limit, page/size, or cursor). Define the parameters once under `components.parameters`, reference them everywhere, and describe the response envelope (items array plus total, next cursor, or links).

## Non-REST and real-time APIs

OpenAPI covers request/response HTTP. For WebSockets, SSE, MQTT, or other event-driven channels, say so and suggest **AsyncAPI** for those parts. Document the HTTP upgrade or subscribe endpoint in OpenAPI and link to the AsyncAPI doc for message formats.

## Minimal complete example

```yaml
openapi: 3.1.0
info:
  title: Task API
  version: 1.0.0
  description: Minimal task management API.
servers:
  - url: http://localhost:8080
tags:
  - name: tasks
paths:
  /tasks:
    get:
      operationId: listTasks
      summary: List tasks
      tags: [tasks]
      parameters:
        - name: done
          in: query
          description: Filter by completion state
          schema: { type: boolean }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: array
                items: { $ref: "#/components/schemas/Task" }
              example:
                - { id: 1, title: Write docs, done: false }
    post:
      operationId: createTask
      summary: Create a task
      tags: [tasks]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: "#/components/schemas/TaskCreate" }
            example: { title: Write docs }
      responses:
        "201":
          description: Created
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Task" }
              example: { id: 2, title: Write docs, done: false }
        "422":
          $ref: "#/components/responses/ValidationError"
  /tasks/{id}:
    parameters:
      - name: id
        in: path
        required: true
        schema: { type: integer, minimum: 1 }
    get:
      operationId: getTask
      summary: Get a task
      tags: [tasks]
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Task" }
        "404":
          $ref: "#/components/responses/NotFound"
components:
  schemas:
    Task:
      type: object
      required: [id, title, done]
      properties:
        id: { type: integer, readOnly: true }
        title: { type: string, minLength: 1, maxLength: 200 }
        done: { type: boolean, default: false }
    TaskCreate:
      type: object
      required: [title]
      additionalProperties: false
      properties:
        title: { type: string, minLength: 1, maxLength: 200 }
    Error:
      type: object
      required: [code, message]
      properties:
        code: { type: string }
        message: { type: string }
  responses:
    NotFound:
      description: Resource not found
      content:
        application/json:
          schema: { $ref: "#/components/schemas/Error" }
          example: { code: NOT_FOUND, message: No task with id 42 }
    ValidationError:
      description: Request body failed validation
      content:
        application/json:
          schema: { $ref: "#/components/schemas/Error" }
          example: { code: VALIDATION_FAILED, message: title is required }
```

## Previewing

Offer one of these, whichever fits the user's setup:

```bash
# Redoc, static HTML
npx @redocly/cli build-docs openapi.yaml -o docs.html

# Swagger UI via Docker
docker run -p 8081:8080 -e SWAGGER_JSON=/spec/openapi.yaml \
  -v "$PWD":/spec swaggerapi/swagger-ui

# Scalar or Bruno/Postman: import openapi.yaml directly
```

For a zero-dependency option, a single HTML page that loads Redoc or Scalar from a CDN and points at `openapi.yaml` works well.

## Converting from other inputs

- **Source code:** read route definitions and handlers, infer schemas from structs/types/validators, and note any field whose type you inferred.
- **curl commands or HAR files:** group by path, generalize IDs into path parameters, and derive schemas from the sample bodies. One sample proves a field exists, not that it is required, so default to optional unless told otherwise.
- **Postman collections:** map folders to tags, requests to operations, and saved responses to examples.
- **Existing Swagger 2.0 or OpenAPI 3.0:** upgrade to 3.1 (`nullable: true` becomes a type union, `example` on schemas becomes `examples`, and so on) and say what changed.

## Pre-delivery checklist

- [ ] Valid OpenAPI 3.1 (lints clean, or re-read carefully if no tooling)
- [ ] Every operation has a unique `operationId`, `summary`, `tags`
- [ ] Every path parameter is `required: true`
- [ ] Success and error responses are documented, not just 200
- [ ] Every `$ref` resolves to something that exists in `components`
- [ ] Every example validates against its schema and matches other endpoints' sample data
- [ ] `required` arrays are accurate
- [ ] Auth scheme is declared and applied
- [ ] Nothing documented that the API does not actually do
- [ ] Assumptions listed for the user

## Output format for your reply

Keep the reply short. Lead with the file, then:

1. **Covered:** endpoints documented (one line)
2. **Assumptions:** anything inferred or guessed
3. **Preview:** the one command that fits their setup

Do not paste the entire spec into chat when you have also saved it as a file.