# Security Audit Report: ruvnet/ruflo (v3.5)

**Date:** 2026-03-13
**Repository:** https://github.com/ruvnet/ruflo
**Auditor:** Automated security analysis
**Verdict:** DO NOT INSTALL - Multiple critical and high-severity vulnerabilities found

---

## Executive Summary

ruflo (formerly claude-flow) is an enterprise AI agent orchestration platform for Claude Code. The audit uncovered **38+ security vulnerabilities** across 7 categories, including **critical command injection**, **arbitrary code execution**, and **authentication bypass**. The project has **18 known vulnerable npm dependencies** (11 high severity).

**Recommendation: Do not install this tool in any environment until the issues below are remediated.**

---

## Severity Distribution

| Severity | Count | Categories |
|----------|-------|------------|
| CRITICAL | 5 | Command injection, arbitrary code execution, auth bypass |
| HIGH | 12 | Hardcoded credentials, unsafe deserialization, privilege escalation, vulnerable deps |
| MEDIUM | 13 | XSS, CORS misconfig, weak crypto, path traversal, SSRF |
| LOW | 8 | Weak randomness, prototype pollution, JSON parse without validation |

---

## Critical Findings

### 1. Arbitrary Command Execution via MCP Tools (CRITICAL)

The `terminal/execute` MCP tool and `ruv-swarm-tools.ts` pass user-supplied commands directly to the OS **without any sanitization or allowlisting**.

- **Impact:** Any MCP client can execute arbitrary shell commands on the host machine.
- **Files:** `v2/src/mcp/`, `ruv-swarm-tools.ts`

### 2. Command Injection via execSync (CRITICAL)

At least **27 instances** of `execSync()` with unsanitized template literal interpolation:

```javascript
// .claude/helpers/github-safe.js:101
execSync(`gh ${args.join(' ')}`, { stdio: 'inherit' });

// v2/bin/automation-executor.js:1411
execSync(`npx claude-flow@alpha memory store "workflow/${this.executionId}/${taskId}" '${resultJson}'`);

// v2/bin/init/index.js:122
execSync(`claude mcp add ${server.name} ${server.command}`, { stdio: 'inherit' });
```

- **Impact:** Shell metacharacters in arguments enable full remote code execution.
- **Files:** `.claude/helpers/github-safe.js`, `v2/bin/swarm.js`, `v2/bin/github.js`, `v2/bin/automation-executor.js`, `v2/bin/init/index.js`, `v2/bin/github/gh-coordinator.js`, and 6+ more files.

### 3. Dynamic Code Execution via eval/AsyncFunction (CRITICAL)

```javascript
// v2/src/consciousness-symphony/consciousness-code-generator.js:316
return eval(`(${newVersion})`);

// v2/examples/browser-dashboard/server-real.js:274
const fn = new AsyncFunction('console', 'sendMCPCommand', code);
```

- **Impact:** Arbitrary JavaScript code execution.

### 4. Authentication Disabled by Default (CRITICAL)

When auth is disabled in `v2/src/mcp/auth.ts`, all requests get `['*']` wildcard permissions, granting full access to every tool.

### 5. Auto-Allow Permission Hooks Bypass (CRITICAL)

`plugin/hooks/hooks.json` uses regex `^mcp__claude-flow__.*$` to auto-approve **all** claude-flow MCP tools without actual validation.

---

## High Severity Findings

### 6. Hardcoded Database Credentials in Source Code

```javascript
// ruflo/src/ruvocal/src/lib/server/database/postgres.ts:24
"postgresql://ruvocal:ruvocal@localhost:5432/ruvocal"
```

### 7. Hardcoded Passwords in Docker Compose Files

Multiple `docker-compose.yml` files ship with plaintext passwords:
- `MONGO_INITDB_ROOT_PASSWORD=password123`
- `GF_SECURITY_ADMIN_PASSWORD=admin123`
- `ME_CONFIG_BASICAUTH_PASSWORD=admin123`
- `POSTGRES_PASSWORD=pass`

**Files:** `v2/examples/05-swarm-apps/rest-api-advanced/docker-compose.yml`, `v2/examples/flask-api-sparc/docker-compose.yml`

### 8. Unsafe Pickle Deserialization (Python)

```python
# v2/examples/ml_foundation/pipelines/data_pipeline.py:298
return pickle.load(f)
```

- **Impact:** Arbitrary code execution from malicious pickle files.

### 9. Privilege Escalation via Agent Spawning

The `agents/spawn` tool creates agents with custom env vars, arbitrary working directories, and **no resource constraints**. Agents inherit parent process privileges.

### 10. Hook System Command Injection

Hook definitions use variable substitution (`$TOOL_INPUT_file_path`) passed directly into shell commands without escaping.

### 11. Vulnerable npm Dependencies (18 total, 11 high)

| Package | Vulnerability | Severity |
|---------|--------------|----------|
| `hono` <= 4.12.6 | Prototype pollution, cookie injection, path traversal, SSE injection | HIGH |
| `tar` <= 7.5.10 | Path traversal, symlink poisoning, hardlink escape (6 CVEs) | HIGH |
| `undici` 7.0.0-7.23.0 | HTTP smuggling, WebSocket DoS, CRLF injection (6 CVEs) | HIGH |
| `@hono/node-server` < 1.19.10 | Auth bypass via encoded slashes | HIGH |
| `flatted` < 3.4.0 | DoS via unbounded recursion | HIGH |
| `express-rate-limit` 8.2.0-8.2.1 | IPv6 bypass of rate limiting | HIGH |
| `esbuild` <= 0.24.2 | Dev server request forgery | MODERATE |
| `file-type` 13.0.0-21.3.1 | Infinite loop DoS, ZIP bomb | MODERATE |

---

## Medium Severity Findings

### 12. DOM-Based XSS

```javascript
// v2/examples/browser-dashboard/dashboard.js:117
li.innerHTML = `<div class="agent-name">${agent.name}</div>`;
```

### 13. SQL Template Injection

```javascript
// .claude/helpers/learning-service.mjs:694
const row = this.db.prepare(`SELECT * FROM ${table} WHERE id = ?`).get(r.patternId);
```

Table name is interpolated directly (currently from constant, but pattern is dangerous).

### 14. CORS Misconfiguration

HTTP transport defaults to `corsOrigins: ['*']` with `credentials: true`.

### 15. Weak Password Hashing

Uses SHA-256 **without salt** instead of bcrypt/argon2.

### 16. Network Exposure

- HTTP/WebSocket transport can listen on `0.0.0.0`
- No HTTPS enforcement by default
- WebSocket connections accepted before authentication
- Health and metrics endpoints are unauthenticated

### 17. Data Exfiltration Risks

Memory export tools, IPFS integration, and OAuth endpoints can send data to external servers without user awareness.

---

## Low Severity Findings

### 18. Weak Randomness for IDs

```javascript
// .claude/helpers/learning-service.mjs:623
const id = `pat_${now}_${Math.random().toString(36).slice(2, 9)}`;
```

### 19. Prototype Pollution

```javascript
// v2/src/consciousness-symphony/consciousness-code-generator.js:326
return Object.assign(this, other);

// v2/examples/04-testing/test-incremental-demo.js:75-86
// deepMerge() without hasOwnProperty check
```

---

## URL Security Note

The URL shared for this repository contained sensitive tokens:
- `mcp_token=eyJ...` - An MCP authentication token (JWT)
- `fbclid=PAR...` - Facebook tracking parameter

**These tokens should be rotated immediately** as they were embedded in the URL and may have been logged or cached.

---

## Final Recommendation

**DO NOT INSTALL** this tool. The combination of:
1. Arbitrary command execution via MCP tools (no sandboxing)
2. 27+ command injection vectors via `execSync`
3. Authentication disabled by default with wildcard permissions
4. Auto-approval of all tool permissions via hooks
5. 18 vulnerable npm dependencies

...makes this tool a significant security risk. It would grant any connected MCP client (or attacker exploiting any of these vectors) **full shell access** to the host machine.

If you still wish to evaluate this tool, do so **only** in an isolated VM/container with no access to sensitive data or networks.
