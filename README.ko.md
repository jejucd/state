# JejuCD 상태 저장소

[English](README.md)

이 저장소는 JejuCD의 원하는 상태를 저장합니다.

Git은 다음 질문에 답합니다.

```text
무엇이 존재해야 하는가
어디에서 실행되어야 하는가
어떤 built-in deployment kind를 사용할 것인가
어떤 단순한 release path를 사용할 것인가
```

Git은 관측된 런타임 상태를 저장하지 않습니다.

JejuCD 데이터베이스는 다음을 저장합니다.

```text
agent heartbeat
배포 이력
노드별 배포 target 결과
배포 로그 메타데이터
관측된 node capability
audit event
user account
token
```

## 저장소 구조

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

`domain_key`는 `payment`, `streaming`, `example` 같은 business/product domain을 뜻합니다. DNS hostname이 아닙니다. DNS hostname은 `[http].domains`에 둡니다.

예시:

```text
apps/payment/api.toml
rbac/domains/payment.toml
```

```toml
[http]
domains = ["api.payment.example.com"]
```

## 상태 파일

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

## 상태 경계

이 저장소에는 선언형 desired state만 저장해야 합니다.

저장할 수 있는 것:

```text
datacenter identity
node scheduling state
node label
app identity
app kind
target node name 또는 label selector
release directory path
작은 deployment default
```

Artifact URL/checksum/version은 CI 또는 manager API가 deployment 요청마다 전달합니다.

저장하면 안 되는 것:

```text
raw secret
node heartbeat timestamp
live systemd status
live Nginx status
설치된 Nginx/systemd/Java/PHP-FPM 상태
deployment log
agent가 발견한 현재 상태
artifact download cache state
user session
```

## Datacenter 상태

Datacenter 파일은 안정적인 배치 정보와 네트워크 컨텍스트를 설명합니다.

예시:

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

Datacenter 상태는 SSH 제어 모델이 아닙니다. Agent가 manager에 join하고, 배포 계획을 노드 로컬에서 적용합니다.

## Node 상태

Node 파일은 앱이 배치될 수 있는 위치를 설명합니다.

Node 상태에는 다음이 포함됩니다.

```text
name
hostname/address
scheduling state
labels
```

Agent version, 감지된 Java version, 설치된 Nginx/systemd/PHP-FPM, 실행 중인 service, 현재 사용 중인 port 같은 live fact는 데이터베이스에 저장합니다.

스케줄링 판단에는 root `[labels]`를 사용합니다. Label은 아주 작은 Kubernetes selector처럼 정확한 key/value match로 동작합니다.

## App 상태

App 파일은 하나의 배포 가능한 단위를 설명합니다.

각 앱은 다음을 정의해야 합니다.

```text
app identity
kind
target rules
release directory paths
```

Target rule은 Git이 소유하는 placement intent와 agent가 관측한 node fact를 분리해야 합니다.

```toml
[target]
nodes = []

[target.labels]
env = "prod"
region = "kr-seoul"
"role.api" = "true"
"runtime.java" = "true"
```

`target.nodes`는 정확한 node allow-list입니다. 비어 있으면 manager는 node의 `[labels]` table이 모든 `target.labels` key/value를 만족하는 node를 선택합니다.

지원하는 app kind는 명시적이어야 합니다.

```text
php-fpm
react-static
rust-service
spring-boot
```

## Simple RBAC

RBAC은 단순해야 합니다. State 파일은 email과 group에 고정된 JejuCD role을 부여합니다. Custom permission string을 각 파일에서 정의하지 않습니다.

Built-in role:

```text
owner    모든 권한, RBAC 변경 포함
admin    app과 deployment 관리
operator deploy, rollback, log 조회
viewer   읽기 전용
```

Access는 넓은 범위에서 좁은 범위로 합쳐집니다.

```text
rbac/global.toml
rbac/domains/{domain_key}.toml
rbac/apps/{domain_key}/{app}.toml
```

`domain_key` 또는 app RBAC 파일이 없으면 parent access를 그대로 상속합니다. 한 user가 여러 role에 매칭되면 가장 높은 role을 사용합니다.

```text
owner > admin > operator > viewer
```

간단한 list만 사용합니다.

```toml
[operators]
emails = ["alice@example.com"]
groups = ["api-operators"]
```

## HTTP Provider 상태

JejuCD state는 raw global Nginx 설정이 아니라 HTTP provider와 route intent를 저장합니다.

`domain_key`는 business/product domain입니다. `[http].domains`는 `api.payment.example.com` 같은 DNS hostname입니다. Repository layout과 RBAC ownership에 쓰는 `domain_key`와 분리합니다.

App state는 다음처럼 표현할 수 있습니다.

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

`provider = "nginx"`이면 agent는 노드 로컬에 다음과 같은 설정을 렌더링합니다.

```text
/etc/nginx/jejucd-enabled/inventory-service.conf
```

JejuCD는 자신이 생성한 include directory만 소유합니다. Global `nginx.conf`, global TLS policy, WAF rule, host-wide Nginx tuning은 소유하지 않습니다.

`provider = "external"`이면 JejuCD는 app만 배포하고 해당 route의 Nginx 설정 생성이나 reload는 수행하지 않습니다.

Nginx reload 전 agent는 반드시 다음을 수행해야 합니다.

```text
candidate config 렌더링
nginx -t 실행
유효한 경우에만 reload
유효하지 않으면 기존 config 유지
failure report
```

## Secrets

Raw secret을 Git에 저장하지 마세요.

Reference를 사용합니다.

```toml
[env]
LOG_LEVEL = "info"
DATABASE_URL = "secret://example/rust-axum-api/DATABASE_URL"
```

MVP에서는 `[env]`에 일반 non-secret 값과 secret reference를 둘 수 있습니다. Raw password, token, private key, database credential은 넣으면 안 됩니다.

Secret backend는 JejuCD encrypted secrets, SOPS, Vault, 또는 다른 provider가 될 수 있습니다. Git state는 secret reference 이름만 저장합니다.

## Target 선택

App은 원하는 node state와 매칭해서 target node를 선택합니다.

Manager는 다음처럼 계산합니다.

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

Manager는 deployment plan을 만들기 전에 다음 충돌을 검증해야 합니다.

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

## Deployment 의미

App 파일을 변경하면 desired deployment state가 변경됩니다.

예시:

```toml
[artifact]
version = "v2.5.2"
```

의미:

```text
JejuCD는 app의 eligible target nodes에 version v2.5.2를 배포해야 합니다.
```

Deployment result, per-node progress, health check output, rollback event는 Git 밖의 observed state로 기록합니다.
