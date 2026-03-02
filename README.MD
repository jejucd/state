# jejucd GitOps Template Repository

Welcome to the official GitOps template for **jejucd**. This repository acts as the single source of truth for infrastructure state and application deployments.

Managing thousands of applications across 500+ nodes requires strict organization to prevent misconfigurations. This repository utilizes an **Atomic Directory Structure** combined with **Domain-Driven Design** to ensure a blast radius of zero. Every node, datacenter, and application is managed as an isolated code object.

---

## 1. Repository Structure

This repository is strictly partitioned into three main layers: Environments (Datacenters), Hardware (Nodes), and Business Domains (Apps).

```text
/gitops-repo
│
├── datacenters/            # The "Environment" Layer (Base Classes)
│   └── dc-seoul.toml       # Global network/SSH for Seoul
│
├── nodes/                  # The "Hardware" Layer (500+ individual files)
│   ├── nodefront01svc.toml # Master node 1
│   └── nodeworker500.toml  # Worker node 500
│
└── apps/                   # The "Business Domain" Layer
    ├── finance/
    │   ├── inventory-service.toml
    │   └── billing-api.toml
    └── logistics/
        └── tracking-system.toml

```

### Architectural Advantages

- **Zero Blast Radius:** Editing `node-1.toml` cannot accidentally impact `node-2.toml`. A syntax error isolates the failure to a single node, rather than compromising the entire fleet.
- **Extreme Searchability:** Infrastructure components can be queried natively using standard OS utilities (e.g., `ls nodes/ | grep front`).
- **Precise Audit Logging:** Git commit history reflects exact, isolated changes (such as a specific node's IP or an application's artifact version) without necessitating reviews of monolithic, multi-thousand-line manifests.

---

## 2. Targeting and Selector Logic

The **jejucd** binary calculates deployment targets using a strict "Double-Lock" intersection logic. This mechanism prevents targeting accidents by separating rigid Enums from flexible Tags.

The targeting logic is defined mathematically as:

$TargetNodes = (CategoryMatch \cap TagMatch) - ExcludeStatus$

### Categorization (Primary Selectors)

These are **First-Class Fields**. Every node must declare these values. Within the `jejucd` binary, these are validated as strict **Enums** to catch typographical errors during the CI/CD pipeline, prior to execution.

| Field              | Type | Enum Examples                           | Description                              |
| ------------------ | ---- | --------------------------------------- | ---------------------------------------- |
| **`role`**         | Enum | `frontend`, `backend`, `db`, `worker`   | High-level infrastructure layer.         |
| **`service_type`** | Enum | `customer-web`, `admin-portal`, `api`   | Specific functional "home" for the app.  |
| **`env`**          | Enum | `prod`, `staging`, `dev`                | Lifecycle environment of the node.       |
| **`status`**       | Enum | `active`, `maintenance`, `provisioning` | Operational state (used for exclusions). |

### Tags (Capability Layer)

Tags are **Dynamic Strings** (`key:value` format) utilized for defining specific node capabilities that fall outside the primary schema.

| Key Prefix          | Example Values           | Description                                     |
| ------------------- | ------------------------ | ----------------------------------------------- |
| **`region:`**       | `seoul`, `tokyo`         | Physical geographic location.                   |
| **`hardware:`**     | `ssd`, `gpu`, `high-mem` | Hardware requirements for specific binaries.    |
| **`legacy_group:`** | `nodefrontsvc`           | Mapping reference to legacy naming conventions. |

---

## 3. Standard Configurations

Below are the approved structural templates for defining resources within this repository.

> **Note:** The `jejucd` parser expects empty strings (`""`) rather than `null` values for unpopulated fields.

### Datacenter Profile (`datacenters/dc-seoul.toml`)

### Node Profile (`nodes/nodefront01svc.toml`)

### Application Profile (`apps/finance/inventory-service.toml`)
