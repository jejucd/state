# JejuCD State Repository

[한국어](README.ko.md)

This repository stores the desired state for JejuCD.

Git answers:

```text
what should exist
where it should run
which built-in deployment kind should be used
which simple release paths should be used
```

Git does not store observed runtime state.

The JejuCD database stores:

```text
agent heartbeats
deployment history
per-node deployment target results
deployment logs metadata
observed node capabilities
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

kinds/
  {kind}.toml

apps/
  {domain_key}/
    {app}.toml

rbac/
  global.toml
  domains/
    {domain_key}.toml
  apps/
    {domain_key}/
      {app}.toml
```

`domain_key` means a business or product domain, such as `payment`, `streaming`, or `example`. It is not a DNS hostname. DNS hostnames live under `[http].domains`.

Example:

```text
apps/payment/api.toml
rbac/domains/payment.toml
```

```toml
[http]
domains = ["api.payment.example.com"]
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
- [example domain RBAC](rbac/domains/example.toml)
- [Spring Boot API RBAC](rbac/apps/example/spring-boot-api.toml)

## State Boundaries

This repository should contain only declarative desired state.

Allowed here:

```text
datacenter identity
node scheduling state
node labels
app identity
app kind
target node names or label selectors
release directory paths
small deployment defaults
```

CI or the manager API should provide artifact URL/checksum/version for each deployment request.

Not allowed here:

```text
raw secrets
node heartbeat timestamps
live systemd status
live Nginx status
installed Nginx/systemd/Java/PHP-FPM status
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
hostname/address
scheduling state
labels
```

Live facts such as agent version, detected Java version, installed Nginx/systemd/PHP-FPM, running services, and current ports in use belong in the database.

Use root `[labels]` for scheduling decisions. Labels are exact key/value matches, similar to a very small Kubernetes selector model.

## App State

App files describe one deployable unit.

Each app should define:

```text
app identity
kind
target rules
release directory paths
```

Target rules should separate Git-owned placement intent from observed node facts:

```toml
[target]
nodes = []

[target.labels]
env = "prod"
region = "kr-seoul"
"role.api" = "true"
"runtime.java" = "true"
```

`target.nodes` is an exact node allow-list. If it is empty, the manager selects nodes where every `target.labels` key/value matches the node's `[labels]` table.

Supported app kinds should be explicit:

```text
php-fpm
react-static
rust-service
spring-boot
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
rbac/domains/{domain_key}.toml
rbac/apps/{domain_key}/{app}.toml
```

Missing `domain_key` or app RBAC files mean "inherit parent access". If a user matches more than one role, the highest role wins:

```text
owner > admin > operator > viewer
```

Use simple lists:

```toml
[operators]
emails = ["alice@example.com"]
groups = ["api-operators"]
```

## HTTP Provider State

JejuCD state stores HTTP provider and route intent, not raw global Nginx configuration.

`domain_key` is the business/product domain. `[http].domains` are DNS hostnames, such as `api.payment.example.com`. They are separate from `domain_key`, which is used for repository layout and RBAC ownership.

App state may say:

```toml
[http]
enabled = true
provider = "nginx"
mode = "reverse_proxy"
domains = ["inventory.example.com"]
path = "/"
listen_port = 80

[http.reverse_proxy]
upstream_address = "127.0.0.1"
upstream_port = 8080
```

When `provider = "nginx"`, the agent renders node-local config such as:

```text
/etc/nginx/jejucd-enabled/inventory-service.conf
```

JejuCD owns only its generated include directory. It does not own global `nginx.conf`, global TLS policy, WAF rules, or host-wide Nginx tuning.

When `provider = "external"`, JejuCD deploys the app but does not render or reload Nginx for that route.

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
  = selected node names, if specified, otherwise all nodes
  + matching environment
  + scheduling enabled
  + matching roles
  + required labels
  - excluded labels
  + required observed capabilities from the database
```

The manager should validate conflicts before creating a deployment plan:

```text
duplicate HTTP hostname
duplicate HTTP hostname/path
duplicate local port on the same node
duplicate systemd service name
duplicate PHP-FPM pool
missing required observed capability
missing Nginx capability when http.provider = "nginx"
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
