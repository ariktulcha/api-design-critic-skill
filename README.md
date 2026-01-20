# API Design Critic

> Transform Claude into a senior API architect that performs expert-level design reviews, catching anti-patterns, security vulnerabilities, and violations of REST best practices.

[![Claude Skill](https://img.shields.io/badge/Claude-Skill-blueviolet?style=for-the-badge&logo=anthropic)](https://claude.ai)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](https://opensource.org/licenses/MIT)

## Overview

API Design Critic is a Claude Skill that analyzes your API designs with the rigor of a senior architect during a design review. It provides actionable feedback with severity levels, concrete fixes, and explanations of *why* patterns matter.

### Supported Formats

- **OpenAPI/Swagger** - YAML and JSON specifications (v2.0 and v3.x)
- **REST Endpoints** - Route definitions, URL patterns, method mappings
- **GraphQL Schemas** - Type definitions, queries, mutations
- **API Documentation** - Markdown docs, code comments, endpoint lists

## What Gets Reviewed

| Category | Checks Performed |
|----------|------------------|
| **Resource Naming** | Plural nouns, kebab-case, no verbs in URLs, proper nesting depth |
| **HTTP Semantics** | Correct method usage, idempotency guarantees, safe method compliance |
| **Response Design** | Consistent envelopes, proper status codes, pagination, HATEOAS links |
| **Error Handling** | Structured errors, appropriate 4xx/5xx usage, correlation IDs |
| **Versioning** | Strategy presence, consistency, deprecation headers |
| **Security** | Auth schemes, sensitive data exposure, rate limiting, CORS |
| **Documentation** | Completeness, examples, schema definitions, auth requirements |

## Sample Review Output

```markdown
# API Design Review: Acme User Service

## Summary
- **Overall Score**: B-
- **Critical Issues**: 2
- **Warnings**: 5
- **Suggestions**: 3

## Critical Issues

### 1. Sensitive Data in URL Parameters
- **Location**: `GET /users?api_key={key}`
- **Problem**: API keys in URLs are logged in server logs, browser history, and CDN caches
- **Impact**: Credential leakage, security vulnerability (OWASP A2)
- **Fix**: Move to `Authorization: Bearer {key}` header

### 2. Missing Rate Limiting
- **Location**: All endpoints
- **Problem**: No rate limit headers or documentation
- **Impact**: Vulnerable to abuse, no client visibility into limits
- **Fix**: Add `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` headers

## Warnings

### 1. Verb in URL
- **Location**: `GET /getUsers`
- **Problem**: URLs should represent resources, not actions
- **Fix**: Change to `GET /users`

## Positive Patterns
- Consistent use of JSON responses
- Good error message clarity
- Proper use of 201 for resource creation
```

## Installation

### Option 1: Skill File (Recommended)

1. Download [`api-design-critic.skill`](https://github.com/ariktulcha/api-design-critic-skill/releases/latest) from Releases
2. Open [claude.ai](https://claude.ai) → Settings → Skills
3. Click "Add Skill" and upload the file

### Option 2: Manual Setup

1. Clone this repository:
   ```bash
   git clone https://github.com/ariktulcha/api-design-critic-skill.git
   ```
2. Copy contents to your Claude skills directory
3. The skill will auto-activate on API-related prompts

## Usage

### Quick Review

```
Review this API for design issues:

POST /createUser
GET /getUserById/{id}
DELETE /users/{id}/remove
```

### OpenAPI Spec Review

```
Analyze this OpenAPI specification for best practice violations:

openapi: 3.0.0
info:
  title: My API
  version: 1.0.0
paths:
  /user/{id}:
    get:
      summary: Get user
      responses:
        200:
          description: Success
```

### Focused Security Audit

```
Perform a security-focused API review of this specification.
Pay special attention to authentication, data exposure, and rate limiting.

[paste your spec]
```

### Full Comprehensive Audit

```
Conduct a comprehensive API design review covering:
- Naming conventions
- HTTP semantics
- Error handling
- Security patterns
- Documentation quality

[paste your spec or endpoints]
```

## Common Patterns Detected

| Anti-Pattern | Issue | Recommendation |
|--------------|-------|----------------|
| `GET /getUsers` | Verb in URL | `GET /users` |
| `GET /user/{id}` | Singular resource | `GET /users/{id}` |
| `POST /users → 200` | Wrong status code | `POST /users → 201` |
| `/users?token=xxx` | Sensitive data in URL | `Authorization` header |
| `/a/{id}/b/{id}/c/{id}` | Deep nesting | Max 2 levels |
| `DELETE → 200 + body` | Unnecessary response | `DELETE → 204` |
| No pagination | Memory/performance risk | `?limit=` + `?offset=` |
| Inconsistent errors | Poor DX | Standardized error schema |

## Skill Structure

```
api-design-critic/
├── SKILL.md              # Core skill instructions and rules
├── common-mistakes.md    # Anti-pattern reference guide
└── rest-patterns.md      # REST best practices reference
```

## Grading Scale

| Grade | Description |
|-------|-------------|
| **A** | Excellent - follows best practices, minor suggestions only |
| **B** | Good - some warnings, no critical issues |
| **C** | Adequate - multiple warnings, potential issues |
| **D** | Poor - critical issues present, needs significant work |
| **F** | Failing - fundamental design problems, security risks |

## Contributing

Contributions are welcome! Areas for improvement:

- **GraphQL patterns** - Query complexity, N+1 detection, schema design
- **gRPC patterns** - Service definitions, streaming, error handling
- **AsyncAPI support** - Event-driven API patterns
- **Additional security checks** - OWASP API Security Top 10

Please open an issue to discuss major changes before submitting a PR.

## License

MIT License - free to use, modify, and distribute.

---

<p align="center">
  Built for <a href="https://claude.ai">Claude</a> by developers who've reviewed too many APIs
</p>
