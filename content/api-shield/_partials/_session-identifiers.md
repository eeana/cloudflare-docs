---
_build:
  publishResources: false
  render: never
  list: never
---

While not strictly required, it is recommended that you configure your {{<glossary-tooltip term_id="session identifier">}}session identifiers{{</glossary-tooltip>}} when getting started with API Shield. When Cloudflare inspects your API traffic for individual sessions, we can offer more tools for visibility, management, and control.

If you are unsure of the session identifiers that your API uses, consult with your development team.

Session identifiers should uniquely identify API clients. A common session identifier for API traffic is the `Authorization` header. When a [JSON Web Token (JWT)](/api-shield/security/jwt-validation/) is used by the API for client authentication, its value may change over time. You can use a claim value inside the JWT such as `sub` or `email` as a session ID to uniquely identify the session over time.