---
name: azure-terraform-import
description: 'Use this to reverse-engineer existing Azure environments into Terraform using Azure CLI discovery and Azure Verified Modules (AVM). It is the preferred approach for importing subscriptions, resource groups, or specific ARM resource IDs; generating Terraform for portal-built environments; creating import blocks; selecting the correct AVM module and version; mapping dependencies; and resolving unexpected plan changes after imports. Apply it even when users describe the problem indirectly, such as “we have no Terraform for this subscription,” “the portal is our source of truth,” “how do we stop drift,” or “can you codify what’s already deployed?” In these cases, the underlying need is an AVM-based import. For existing Azure resources, prefer this approach over manually writing azurerm_* resources.'
---

# Azure Terraform Import

Convert existing Azure infrastructure into maintainable Terraform code using Azure resource discovery and Azure Verified Modules (AVM).

## When to Use This Skill

Use this skill when the user asks to:

- Import existing Azure resources into Terraform
- Prefer AVM modules over handwritten `azurerm_*` resources for existing infrastructure
- Recreate infrastructure from a subscription, resource group, or set of resource IDs
- Troubleshoot Terraform imports, drift, or unexpected plan changes
- Map dependencies between discovered Azure resources
- Select and implement the appropriate AVM modules

## Prerequisites

- Azure CLI installed and authenticated (`az login`)
- Access to the target subscription or resource group
- Access to Terraform Registry and AVM resources
- Terraform CLI installed

## Inputs

At least one of `subscription-id`, `resource-group-name`, or `resource-id` is required.

| Parameter | Required | Default | Description |
|---|---|---|---|
| `subscription-id` | No | Active CLI context | Azure subscription used to discover resources at the subscription scope and set the Azure CLI context. |
| `resource-group-name` | No | None | Azure resource group used to discover resources within the resource-group scope. |
| `resource-id` | No | None | One or more Azure ARM resource IDs used to discover resources at the specific-resource scope. |


## Workflow

### 1. Define the Discovery Scope (Required)

Before running any discovery commands, identify one of the following scopes:

- Subscription scope: `<subscription-id>`
- Resource group scope: `<resource-group-name>`
- Specific resource scope: one or more Azure ARM `<resource-id>` values

Scope Rules:

- Treat Azure ARM resource IDs (for example, `/subscriptions/.../providers/...`) as Azure resource identifiers, not local file paths.
- Use resource IDs only with Azure CLI commands that support `--ids`.
- Never pass resource IDs to file operations such as `cat`, `ls`, `file readers`, or `path-based` searches unless the user explicitly identifies them as local files.
- If a valid scope has already been provided, do not request additional scope information unless required to resolve an error.
- Avoid follow-up questions that can be answered from the supplied scope.

If no scope is provided, request it before proceeding.

### 2. Authenticate and Set Azure Context

Run only the commands required for the selected scope.

Subscription scope:

```bash
az login
az account set --subscription <subscription-id>
az account show --query "{subscriptionId:id, name:name, tenantId:tenantId}" -o json
```

Expected output:

```json
{
  "subscriptionId": "<subscription-id>",
  "name": "<subscription-name>",
  "tenantId": "<tenant-id>"
}
```

Resource group or specific-resource scope:

```bash
az login
```

- `az account set` is optional if the active subscription context is already correct.
- For resource-specific imports, prefer direct `--ids` queries and avoid unnecessary subscription-wide discovery.

### 3. Discover Existing Resources

Collect resource metadata from Azure based on the selected scope.

```bash
# Subscription scope
az resource list --subscription <subscription-id> -o json

# Resource group scope
az resource list --resource-group <resource-group-name> -o json

# Specific resource scope
az resource show --ids <resource-id-1> <resource-id-2> ... -o json
```

Capture all relevant resource information, including:

- id
- type
- name
- location
- tags
- properties

This discovery data becomes the source of truth for Terraform generation.

### 4. Analyze Dependencies

Before generating Terraform, build a dependency model from the discovered resources.

Identify:
- Parent-child relationships (for example, `NIC → Subnet → VNet`)
- References between resources in properties
- Required Terraform creation and import order
- Shared infrastructure dependencies

The resulting dependency graph should ensure imports and module composition accurately reflect the deployed environment.

### 5. Generate Discovery Documentation

Create a docs/ directory in the project root and save the discovery outputs.

#### Required Artifacts

##### docs/exported-resources.json
- Complete inventory of discovered Azure resources
- Resource metadata
- Dependency mappings
- Cross-resource references

##### docs/exported-architecture.md
- Human-readable architecture summary
- Resource hierarchy and relationships
- Dependency overview
- Key infrastructure components and design observations

These artifacts serve as both audit documentation and the foundation for AVM-based Terraform generation.

### 6. Select Azure Verified Modules (AVM)

> **Required:** Use Azure Verified Modules whenever a suitable AVM module exists. Only use native `azurerm_*` resources when no AVM module is available, and document the reason for the exception.

#### Identify the Correct Module

For each discovered Azure resource:

1. Find the corresponding AVM module.
2. Select the latest compatible module version.
3. Record the module source and version before generating Terraform code.

##### Module Discovery Sources

**Terraform Registry**

- Search for `avm <resource-name>`
- Filter results by the **Partner** publisher tag
- Example: `avm storage account`

**Official AVM Module Indexes**

The AVM indexes provide the authoritative list of available modules:

- Terraform Resource Modules  
  `https://raw.githubusercontent.com/Azure/Azure-Verified-Modules/refs/heads/main/docs/static/module-indexes/TerraformResourceModules.csv`

- Terraform Pattern Modules  
  `https://raw.githubusercontent.com/Azure/Azure-Verified-Modules/refs/heads/main/docs/static/module-indexes/TerraformPatternModules.csv`

- Terraform Utility Modules  
  `https://raw.githubusercontent.com/Azure/Azure-Verified-Modules/refs/heads/main/docs/static/module-indexes/TerraformUtilityModules.csv`

> These files always reference the latest content on the main branch. Use a release tag if a point-in-time version is required.

#### Gather Module Documentation

If module metadata is not available locally, retrieve it from:

- Terraform Registry  
  `https://registry.terraform.io/modules/Azure/<module>/azurerm/latest`

- GitHub Repository  
  `https://github.com/Azure/terraform-azurerm-avm-res-<service>-<resource>`

The repository `README.md` is the primary source of module documentation and implementation guidance.

---

### 6.1 Read the Module README Before Writing Code

> **Mandatory:** Never write AVM-based Terraform from memory or from raw `azurerm` provider knowledge.

For every selected module, review the module README before generating HCL.

Retrieve documentation from either:

```text
https://raw.githubusercontent.com/Azure/terraform-azurerm-avm-res-<service>-<resource>/refs/heads/main/README.md
```

Or from the downloaded module source:

```bash
cat .terraform/modules/<module_key>/README.md
```

#### Capture the Following Information

##### Required Inputs

Identify all required inputs and determine which resources are managed internally by the module.

Examples may include:

- Network interfaces
- Virtual machine extensions
- Public IPs
- Subnets

Do not create separate modules for resources that are explicitly managed by a parent module.

##### Optional Inputs

Document:

- Variable names
- Supported types
- Accepted values
- Default behavior

Do not assume AVM variables use the same names or structures as the underlying `azurerm` provider.

##### Usage Patterns

Review examples to understand:

- Resource group identifiers (`parent_id` vs `resource_group_name`)
- Nested resource definitions
- Map structures
- Provider-specific conventions

#### AVM Implementation Rules

For every selected module:

- Determine resource ownership from **Required Inputs**
- Determine accepted parameters from **Optional Inputs** and `variables.tf`
- Follow README examples for identifier formats and input structures
- Never infer arguments from native `azurerm_*` resources

---

### 7. Generate Terraform Configuration

Generate Terraform code using the selected AVM modules and dependency model.

#### Required Files

Create:
- `providers.tf`
- `main.tf`
- `variables.tf`
- `outputs.tf`
- `terraform.tfvars.example`

#### Pin Module Versions

Always specify a fixed module version:

```hcl
module "example" {
  source  = "Azure/<module>/azurerm"
  version = "<latest-compatible-version>"
}
```

---

### 7.1 Inspect Module Source Before Writing Import Blocks

> **Mandatory:** Never construct Terraform import addresses from memory.

After downloading modules with `terraform init`, inspect the module source to determine the exact resource addresses.

#### Step A: Identify Resource Types and Labels

```bash
grep "^resource" .terraform/modules/<module_key>/main*.tf
```

Determine:

- Provider type (`azurerm` vs `azapi`)
- Resource labels
- Actual Terraform addresses

#### Step B: Discover Nested Modules

```bash
grep "^module" .terraform/modules/<module_key>/main*.tf
```

If resources are deployed through child modules, imports must include the complete nested path.

Example:

```text
module.<root_module>.module.<child_module>["<key>"].<resource_type>.<label>[<index>]
```

#### Step C: Check `count` and `for_each`

```bash
grep -n "count\|for_each" .terraform/modules/<module_key>/main*.tf
```

Import addresses must match implementation details:

- `count` resources require numeric indices such as `[0]`
- `for_each` resources require string keys

#### Common Import Address Patterns

The following patterns are examples only and must be verified against the downloaded module source code.

| Resource Type | Example Import Address |
|---------------|------------------------|
| Virtual Network | `module.<vnet>.azapi_resource.vnet` |
| Subnet | `module.<vnet>.module.subnet["<name>"].azapi_resource.subnet[0]` |
| Linux VM | `module.<vm>.azurerm_linux_virtual_machine.this[0]` |
| VM NIC | `module.<vm>.azurerm_network_interface.virtualmachine_network_interfaces["<nic>"]` |
| VM Extension | `module.<vm>.module.extension["<extension>"].azurerm_virtual_machine_extension.this` |

Always derive the final address from module source code, not from examples.

---

### 7.2 Reconcile Live Configuration with Module Defaults

> **Mandatory:** Imported infrastructure should not drift because module defaults differ from live Azure settings.

For each discovered resource:

1. Retrieve detailed live properties from Azure.
2. Compare live values against AVM defaults in `variables.tf`.
3. Explicitly configure any value that differs.

#### Common Sources of Drift

- Timeouts and idle settings
- Network policy configuration
- SKU and allocation settings
- Availability zones
- Storage redundancy and replication settings
- Database configuration options

Example property checks:

```bash
az network public-ip show \
  --ids <resource_id> \
  --query "{idleTimeout:idleTimeoutInMinutes,sku:sku.name,zones:zones}" \
  -o json
```

```bash
az network vnet subnet show \
  --ids <resource_id> \
  --query "{privateEndpointPolicies:privateEndpointNetworkPolicies,delegations:delegations}" \
  -o json
```

Do not rely solely on `az resource list`, as it may omit nested or computed properties.

---

### 8. Validate the Generated Terraform

Run:

```bash
terraform init
terraform fmt -recursive
terraform validate
terraform plan
```

#### Success Criteria

The configuration is considered valid when:

- No syntax errors are reported
- Validation completes successfully
- Dependencies are resolved correctly
- The plan accurately reflects the discovered environment

The import should not be considered complete until the plan shows:

- **0 destroys**
- **0 unintended updates**

Telemetry-related resources created by modules are acceptable if expected.

---

## Troubleshooting

| Issue | Likely Cause | Recommended Action |
|---------|-------------|-------------------|
| Azure CLI authorization errors | Incorrect tenant, subscription, or RBAC permissions | Re-authenticate and verify access |
| Discovery returns no resources | Incorrect scope | Validate subscription, resource group, or resource IDs |
| No AVM module available | Module does not exist yet | Use `azurerm_*` and document the exception |
| `terraform validate` fails | Missing variables or dependencies | Add required inputs and dependency references |
| Unknown module argument | AVM variable differs from provider argument | Check README and `variables.tf` |
| Import address not found | Incorrect provider type, label, nesting, or index | Inspect downloaded module source |
| Unexpected updates in plan | Live values differ from module defaults | Explicitly set live values in configuration |
| Child-resource ownership issues | Resource managed by parent module | Model resource through the parent module inputs |
| Nested import failures | Missing module path, map key, or index | Build the complete address from module source |
| ARM IDs treated as file paths | Incorrect handling of resource identifiers | Use Azure CLI `--ids` arguments only |

---

## Response Requirements

When presenting results, include:

1. Discovery scope used
2. Documentation and discovery files created
3. Resource types identified
4. Selected AVM modules and versions
5. Terraform files generated or modified
6. Validation command results
7. Outstanding gaps or required user input

---

## Agent Rules

- Do not proceed without a defined scope.
- Do not skip dependency analysis before code generation.
- Prefer AVM modules whenever available.
- Explicitly justify every non-AVM implementation.
- Read each AVM module README before writing code.
- Determine child-resource ownership from module documentation, not assumptions.
- Inspect downloaded module source before creating import blocks.
- Treat ARM resource IDs as Azure identifiers, never local file paths.
- Avoid unnecessary scope-related prompts when valid scope information is already available.
- Do not declare the import complete until Terraform plan results show no unintended changes.

## References

- Azure Verified Modules Index (Terraform)  
  `https://github.com/Azure/Azure-Verified-Modules/tree/main/docs/static/module-indexes`

- Terraform AVM Registry Namespace  
  `https://registry.terraform.io/namespaces/Azure`