# JejuCD State Repository

[한국어](README.ko.md)

This repository stores the desired state for JejuCD.

Git answers:

```text
what should exist
which app version should run
where it should run
which runtime should be used
which systemd service should exist
which Nginx route should exist
which health check defines success
```

Git does not store observed runtime state.

The JejuCD database stores:

```text
agent heartbeats
current deployed versions reported by nodes
deployment history
deployment logs metadata
locks
audit events
user accounts
tokens
```

## Repository Layout

```text
datacenters/
  {datacenter}.toml

nodes/
  {node}.toml

apps/
  {domain}/
    {app}.toml

rbac/
  global.toml
  domains/
    {domain}.toml
  apps/
    {domain}/
      {app}.toml
```

## State Files

Datacenters:

- [kr-seoul-01](datacenters/kr-seoul-01.toml)

Nodes:

- [kr-seoul-m-01](nodes/kr-seoul-m-01.toml)
- [nodefront01svc](nodes/nodefront01svc.toml)

Example apps:

- [PHP Shop](apps/example/php-shop.toml)
- [React Admin](apps/example/react-admin.toml)
- [Rust Axum API](apps/example/rust-axum-api.toml)
- [Spring Boot API](apps/example/spring-boot-api.toml)

RBAC:

- [Global RBAC](rbac/global.toml)
- [example.com RBAC](rbac/domains/example.com.toml)
- [Spring Boot API RBAC](rbac/apps/example/spring-boot-api.toml)

## State Boundaries

This repository should contain only declarative desired state.

Allowed here:

```text
datacenter identity
node placement labels
node scheduling state
node capacity policy
app identity
artifact URL/checksum/version
target selection rules
runtime intent
systemd service intent
Nginx route intent
health check intent
rollback policy
secret references
```

Not allowed here:

```text
raw secrets
node heartbeat timestamps
live systemd status
live Nginx status
deployment logs
current state discovered by agents
artifact download cache state
user sessions
```

## Datacenter State

Datacenter files describe stable placement and network context.

Example:

```toml
[datacenter]
id = "kr-seoul-01"
name = "Seoul Central Hub"
region = "ap-northeast-2"
status = "active"

[network]
gateway = "10.0.0.1"
dns = ["8.8.8.8"]
ntp = ["pool.ntp.org"]
proxy_url = ""
```

Datacenter state is not an SSH control model. Agents join the manager and apply deployment plans locally.

## Node State

Node files describe where apps are allowed to land.

Node state includes:

```text
name
hostname
environment
datacenter
scheduling state
roles
labels
tags
port range
max app count
```

Live facts such as agent version, detected Java version, running services, and current ports in use belong in the database.

## App State

App files describe one deployable unit.

Each app should define:

```text
app identity
kind
artifact
target rules
runtime
service
http route
health check
deployment strategy
rollback behavior
environment values
```

Supported app kinds should be explicit:

```text
spring-boot
rust-service
static-site
php-app
systemd-service
```

## Simple RBAC

RBAC should stay boring. State files grant fixed JejuCD roles to emails and groups; they should not define custom permission strings.

Built-in roles:

```text
owner    everything, including RBAC changes
admin    manage apps and deployments
operator deploy, rollback, and read logs
viewer   read only
```

Access is merged from broad to narrow:

```text
rbac/global.toml
rbac/domains/{domain}.toml
rbac/apps/{domain}/{app}.toml
```

Missing domain or app RBAC files mean "inherit parent access". If a user matches more than one role, the highest role wins:

```text
owner > admin > operator > viewer
```

Use simple lists:

```toml
[operators]
emails = ["alice@example.com"]
groups = ["api-operators"]
```

## Nginx State

JejuCD state stores Nginx route intent, not raw global Nginx configuration.

App state may say:

```text
domain = inventory.example.com
mode = reverse_proxy
upstream = 127.0.0.1:8080
```

The agent renders node-local config such as:

```text
/etc/nginx/jejucd-enabled/inventory-service.conf
```

JejuCD owns only its generated include directory. It does not own global `nginx.conf`, global TLS policy, WAF rules, or host-wide Nginx tuning.

Before every Nginx reload, the agent must:

```text
render candidate config
run nginx -t
reload only if valid
keep old config if invalid
report failure
```

## Secrets

Do not store raw secrets in Git.

Use references:

```toml
[env]
LOG_LEVEL = "info"
DATABASE_URL = "secret://example/rust-axum-api/DATABASE_URL"
```

For MVP, `[env]` may contain normal non-secret values and secret references. It should not contain raw passwords, tokens, private keys, or database credentials.

Secret backends may be JejuCD encrypted secrets, SOPS, Vault, or another configured provider. The Git state only names the secret reference.

## Target Selection

Apps target nodes by matching desired node state.

The manager calculates:

```text
eligible nodes
  = selected node names, if specified
  + nodes with scheduling enabled
  + matching environment
  + matching roles
  + required tags
  - excluded tags
```

The manager should validate conflicts before creating a deployment plan:

```text
duplicate domain
duplicate domain/path
duplicate local port on the same node
duplicate systemd service name
duplicate PHP-FPM pool
missing required runtime capability
missing Nginx capability for HTTP apps
invalid artifact checksum format
```

## Deployment Meaning

Changing an app file changes desired deployment state.

Example:

```toml
[artifact]
version = "v2.5.2"
```

means:

```text
JejuCD should deploy version v2.5.2 to the app's eligible target nodes.
```

Deployment result, per-node progress, health check output, and rollback events are recorded outside Git as observed state.
