# cloud-infra-to-iaac

A Claude Code skill that reverse-engineers **live Azure infrastructure into maintainable Terraform**, built on Azure CLI discovery and [Azure Verified Modules (AVM)](https://github.com/Azure/Azure-Verified-Modules).

The skill lives at [`.claude/azure-terraform-import/SKILL.md`](.claude/azure-terraform-import/SKILL.md) and is invoked as `azure-terraform-import`.

---

## What it does

You point it at something that already exists in Azure: a subscription, a resource group, or a handful of ARM resource IDs. It discovers what's there, builds a dependency model, writes audit documentation, selects the right AVM module for each resource type, generates the Terraform, derives the exact `import {}` addresses from the downloaded module source, reconciles live property values against module defaults, and validates until `terraform plan` is clean.

The goal is not "some Terraform that looks right." It is Terraform that **adopts** the existing resources without recreating them, and where `terraform plan` shows no destroys and no unintended updates.

## When to use it

- You inherited an environment built by hand in the portal and need it under version control
- You want to codify a subscription, resource group, or set of resource IDs as-is, without a rebuild
- You need to stop configuration drift on resources nobody has a source of truth for
- You are troubleshooting a Terraform import, or a plan that shows unexpected changes after one
- You need dependencies mapped between discovered Azure resources
- You need the appropriate AVM modules selected and implemented

## Prerequisites

| Requirement | Why |
|---|---|
| Azure CLI, authenticated (`az login`) | All discovery runs through `az` |
| Access to the target subscription or resource group | Discovery fails without the right RBAC |
| Terraform CLI | `init` / `validate` / `plan`, and to download module source for address derivation |
| Access to Terraform Registry and AVM resources | Module selection and README lookups |

## Inputs

At least **one** scope is required. Everything else is optional.

| Parameter | Required | Default | Description |
|---|---|---|---|
| `subscription-id` | No | Active CLI context | Discovery at the subscription scope, and setting the Azure CLI context |
| `resource-group-name` | No | None | Discovery within the resource-group scope |
| `resource-id` | No | None | One or more ARM resource IDs, for discovery at the specific-resource scope |

ARM resource IDs are treated strictly as **Azure identifiers**, never as local file paths; they are only ever passed to Azure CLI commands that support `--ids`.

---

## Step-by-step functionality

### 1. Define the discovery scope (required)

The skill identifies exactly one scope before running anything, and requests it if none was supplied. Once a valid scope is present it stops asking; there is no re-prompting for a subscription when resource IDs were already given, and no follow-up questions the supplied scope already answers.

### 2. Authenticate and set Azure context

Runs only what the chosen scope needs:

```bash
az login
az account set --subscription <subscription-id>
az account show --query "{subscriptionId:id, name:name, tenantId:tenantId}" -o json
```

For resource-group or specific-resource scope, `az account set` is optional when the active context is already correct, and `--ids` queries are preferred over subscription-wide discovery.

### 3. Discover existing resources

```bash
az resource list --subscription <subscription-id> -o json           # subscription scope
az resource list --resource-group <resource-group-name> -o json     # resource group scope
az resource show --ids <resource-id-1> <resource-id-2> ... -o json  # specific resources
```

Captures `id`, `type`, `name`, `location`, `tags` and `properties` for each resource. This discovery data becomes the source of truth for everything generated downstream.

### 4. Analyze dependencies

Before any Terraform is written, a dependency model is built from the discovered resources:

- Parent-child relationships (`NIC → Subnet → VNet`)
- References between resources buried in `properties`
- Required Terraform creation and import order
- Shared infrastructure dependencies

The resulting graph is what makes imports and module composition reflect the deployed environment rather than a guess at it.

### 5. Generate discovery documentation

A `docs/` directory is created in the project root holding two artefacts:

| Artefact | Contents |
|---|---|
| `docs/exported-resources.json` | Complete inventory of discovered resources, metadata, dependency mappings, cross-resource references |
| `docs/exported-architecture.md` | Human-readable architecture summary: resource hierarchy, dependency overview, key components and design observations |

These serve as audit documentation in their own right, as well as the foundation for the generated Terraform.

### 6. Select Azure Verified Modules

> **Required:** use an AVM module wherever a suitable one exists. Native `azurerm_*` resources are only for cases where no AVM module is available, and the reason is documented.

For each discovered resource: find the corresponding AVM module, select the latest compatible version, and record the source and version *before* generating code.

**Module discovery sources**

- **Terraform Registry**: search `avm <resource-name>`, filter by the **Partner** publisher tag (for example, `avm storage account`)
- **Official AVM indexes** (authoritative, latest content on `main`):
  - [`TerraformResourceModules.csv`](https://raw.githubusercontent.com/Azure/Azure-Verified-Modules/refs/heads/main/docs/static/module-indexes/TerraformResourceModules.csv)
  - [`TerraformPatternModules.csv`](https://raw.githubusercontent.com/Azure/Azure-Verified-Modules/refs/heads/main/docs/static/module-indexes/TerraformPatternModules.csv)
  - [`TerraformUtilityModules.csv`](https://raw.githubusercontent.com/Azure/Azure-Verified-Modules/refs/heads/main/docs/static/module-indexes/TerraformUtilityModules.csv)
- **Module documentation**: the Registry page (`https://registry.terraform.io/modules/Azure/<module>/azurerm/latest`) or the GitHub repository (`https://github.com/Azure/terraform-azurerm-avm-res-<service>-<resource>`), whose `README.md` is the primary implementation guidance

### 6.1 Read the module README before writing code

> **Mandatory:** never write AVM-based Terraform from memory, or from raw `azurerm` provider knowledge.

```text
https://raw.githubusercontent.com/Azure/terraform-azurerm-avm-res-<service>-<resource>/refs/heads/main/README.md
```

Or, once modules are downloaded:

```bash
cat .terraform/modules/<module_key>/README.md
```

Three things are captured before any HCL is written:

| From | What to extract | Why it matters |
|---|---|---|
| **Required Inputs** | Which resources the module manages internally (NICs, VM extensions, public IPs, subnets) | Creating a separate module for a parent-owned resource is the classic AVM failure |
| **Optional Inputs** | Variable names, supported types, accepted values, default behaviour | AVM variable names and structures frequently differ from the underlying provider's |
| **Usage patterns** | Resource group identifiers (`parent_id` vs `resource_group_name`), nested resource definitions, map structures, provider conventions | Identifier style varies per module, including between AzAPI- and azurerm-backed ones |

The rules that follow: determine ownership from Required Inputs, determine accepted parameters from Optional Inputs and `variables.tf`, follow the README examples for identifier formats, and never infer arguments from native `azurerm_*` resources.

### 7. Generate Terraform configuration

Generated from the selected modules and the dependency model:

| File | Contents |
|---|---|
| `providers.tf` | Provider configuration and version constraints |
| `main.tf` | AVM module blocks with explicit dependencies |
| `variables.tf` | Environment-specific values |
| `outputs.tf` | Key IDs and endpoints |
| `terraform.tfvars.example` | Placeholder values |

Module versions are always pinned to a fixed version, never floated.

### 7.1 Inspect module source before writing import blocks

> **Mandatory:** never construct import addresses from memory.

After `terraform init` downloads the modules, the real addresses are derived from the downloaded source:

```bash
grep "^resource" .terraform/modules/<module_key>/main*.tf            # provider type (azurerm vs azapi), labels
grep "^module"   .terraform/modules/<module_key>/main*.tf            # nested modules, so the full path
grep -n "count\|for_each" .terraform/modules/<module_key>/main*.tf   # [0] index vs string key
```

Resources reached through a child module need the complete nested path:

```text
module.<root_module>.module.<child_module>["<key>"].<resource_type>.<label>[<index>]
```

`count`-based resources require a numeric index such as `[0]`; `for_each`-based resources require string keys.

Example patterns, to be verified against the downloaded source every time:

| Resource type | Example import address |
|---|---|
| Virtual Network | `module.<vnet>.azapi_resource.vnet` |
| Subnet | `module.<vnet>.module.subnet["<name>"].azapi_resource.subnet[0]` |
| Linux VM | `module.<vm>.azurerm_linux_virtual_machine.this[0]` |
| VM NIC | `module.<vm>.azurerm_network_interface.virtualmachine_network_interfaces["<nic>"]` |
| VM Extension | `module.<vm>.module.extension["<extension>"].azurerm_virtual_machine_extension.this` |

### 7.2 Reconcile live configuration with module defaults

> **Mandatory:** imported infrastructure should not drift because a module default differs from the live Azure setting.

For each discovered resource: retrieve the detailed live properties, compare them against the AVM defaults in `variables.tf`, and explicitly configure anything that differs.

Common sources of drift:

- Timeouts and idle settings
- Network policy configuration
- SKU and allocation settings
- Availability zones
- Storage redundancy and replication settings
- Database configuration options

Properties are pulled with targeted `az ... show` calls, because `az resource list` may omit nested or computed values:

```bash
az network public-ip show \
  --ids <resource_id> \
  --query "{idleTimeout:idleTimeoutInMinutes,sku:sku.name,zones:zones}" \
  -o json

az network vnet subnet show \
  --ids <resource_id> \
  --query "{privateEndpointPolicies:privateEndpointNetworkPolicies,delegations:delegations}" \
  -o json
```

### 8. Validate the generated Terraform

```bash
terraform init
terraform fmt -recursive
terraform validate
terraform plan
```

**Success criteria:** no syntax errors, validation completes, dependencies resolve, and the plan accurately reflects the discovered environment.

**Definition of done:** the plan shows **0 destroys** and **0 unintended updates**. Telemetry resources created by the modules are acceptable when expected.

---

## What you get back

Every run reports:

1. Discovery scope used
2. Documentation and discovery files created
3. Resource types identified
4. Selected AVM modules and versions
5. Terraform files generated or modified
6. Validation command results
7. Outstanding gaps or required user input

## Agent rules

The constraints the skill holds itself to, and the reasoning behind each:

- **No scope, no start.** Discovery against the wrong subscription is worse than no discovery.
- **Dependency analysis before code generation.** Import order and module composition both depend on it.
- **AVM first, exceptions justified.** Every non-AVM implementation is explicitly documented.
- **README before HCL.** Required Inputs decide child-resource ownership; guessing produces duplicated resources and provider errors.
- **Ownership from documentation, not assumption.** Whether a NIC, extension or public IP is standalone is a per-module fact.
- **Inspect module source before import blocks.** Provider type, nesting and `count` vs `for_each` all vary by module and version.
- **ARM IDs are Azure identifiers, never file paths.** They belong in `az --ids`, never in `cat`, `ls` or a glob.
- **No unnecessary scope prompts.** Ask again only when a command genuinely fails for want of context.
- **Not complete until the plan is clean.** No unintended changes, or it isn't done.

## Troubleshooting

| Issue | Likely cause | Recommended action |
|---|---|---|
| Azure CLI authorization errors | Incorrect tenant, subscription, or RBAC permissions | Re-authenticate and verify access |
| Discovery returns no resources | Incorrect scope | Validate the subscription, resource group, or resource IDs |
| No AVM module available | Module does not exist yet | Use `azurerm_*` and document the exception |
| `terraform validate` fails | Missing variables or dependencies | Add the required inputs and dependency references |
| Unknown module argument | AVM variable differs from the provider argument | Check the README and `variables.tf` |
| Import address not found | Incorrect provider type, label, nesting, or index | Inspect the downloaded module source |
| Unexpected updates in plan | Live values differ from module defaults | Explicitly set the live values in configuration |
| Child-resource ownership issues | Resource is managed by a parent module | Model it through the parent module's inputs |
| Nested import failures | Missing module path, map key, or index | Build the complete address from module source |
| ARM IDs treated as file paths | Incorrect handling of resource identifiers | Use Azure CLI `--ids` arguments only |

## References

- [Azure Verified Modules index (Terraform)](https://github.com/Azure/Azure-Verified-Modules/tree/main/docs/static/module-indexes)
- [Terraform AVM Registry namespace](https://registry.terraform.io/namespaces/Azure)
