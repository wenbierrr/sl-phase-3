# Intent

Deploy LibreChat with Kubernetes MCP server in alignment to svc design, so that deployment bottlenecks can be reduced and incident response time can be shortened.


# Outcome

Empower every DE with the context and/or knowledge they lack to troubleshoot cluster and deployment issues, while ensuring the tool cannot be misused or adversely affect cluster operations


# Impact

| Affected group | Cost of inaction |
|---|---|
| Duty engineers | Longer troubleshooting time to piece together scattered information & make sense of it |
| Starforge | Provide assistance with escalations that could have otherwise been solved with the tool |
| Application teams | Longer deployment times in the event of errors |
| The platform | Identifying the root cause can be time consuming, so incidents and outages last longer (especially for incidents never seen before) |


# Out of scope

| Item | Note |
|---|---|
| GitLab MCP | Will be revisited if git access is needed |
| Installation Modality | Will be revisited if ONLY ONE centralised librechat is needed for all clusters |
| LLM hosting and model selection | Out of scope as mentioned  in RFC-1. |

# Key Tasks
- mechanism of rate limiting
- producing relevant docs
- bring into stg for soak in
- roll out to different clusters

# Acceptance

- [ ] **1x service design**, covering the architecture, the read-only ClusterRole, authentication and audit, the Kube API rate limit, the system prompt.
- [ ] **1x installation guide** for the Kubernetes MCP server, covering its install, RBAC and rate-limit configuration, and how LibreChat connects to it
- [ ] **1x user guide**, covering how to ask (name the namespace; describe symptoms, not a diagnosis), how to read the answer format, and what the tool cannot see.
- [ ] **Deploy in alignment to service design**
