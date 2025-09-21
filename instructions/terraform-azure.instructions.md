---
description: 'Guidelines for generating modern Terraform code for Azure'
applyTo: '**/*.tf'
---


# Terraform for Azure Solutions - Best Practices Guide

## 1. Scope

This applies to writing Terraform solution code for Azure. Solution code makes use of existing Terraform modules to create the components required for a specific application or service.

If the user is creating a new module, please refer to the [Azure Verified Modules for Terraform](instructions/azure-verified-modules-terraform.instructions.md) instructions.

Whilst not scoped to modules, AVM non-functional requirements (TFNFR#) that are relevant to creating solutions have been referenced, to guide consistency.

**Specification-First Approach**
When no clear specification exists in the repository, recommend using a specification driven approach with an appropriate chatmode to ensure that functional & non-functional requirements are captured before attempting to write the code for the solution. Use [terraform-planning chatmode](chatmodes/terraform-planning.chatmode.md) for planning and [terraform-implement chatmode](chatmodes/terraform-implement.chatmode.md) for implementation.

## 2. Common Anti-Patterns to Avoid

**Configuration:**

- MUST NOT hardcode values that should be parameterized
- SHOULD NOT use `terraform import` as a regular workflow pattern
- SHOULD avoid complex conditional logic that makes code hard to understand
- MUST NOT use `local-exec` provisioners unless absolutely necessary

**Security:**

- MUST NEVER store secrets in Terraform files or state
- MUST avoid overly permissive IAM roles or network rules
- MUST NOT disable security features for convenience
- MUST NOT use default passwords or keys

**Operational:**

- MUST NOT apply Terraform changes directly to production without testing
- MUST avoid making manual changes to Terraform-managed resources
- MUST NOT ignore Terraform state file corruption or inconsistencies
- MUST NOT run Terraform from local machines for production

---

## 3. Organize Code Cleanly

Structure Terraform configurations with logical file separation:

- Use `main.tf` for resources
- Use `variables.tf` for inputs
- Use `outputs.tf` for outputs
- Use `terraform.tf` for provider configurations
- Use `locals.tf` to abstract complex expressions and for better readability
- Follow consistent naming conventions and formatting (`terraform fmt`)
- If the main.tf or variables.tf files grow too large, split them into multiple files by resource type or function (e.g., `main.networking.tf`, `main.storage.tf` - move equivalent variables to `variables.networking.tf`, etc.)

Use snake casing for variables and module names.

This makes the code easy to navigate and maintain.

## 4. Use Azure Verified Modules (AVM)

**Reference:** See [Azure Verified Modules for Terraform](instructions/azure-verified-modules-terraform.instructions.md) for module usage, style, and non-functional requirements. Align your solution with AVM patterns for maintainability and supportability. If you are creating a new module, follow the AVM instructions strictly.

Any significant resource should use an AVM if available. AVMs are designed to be aligned to the Well Architected Framework, are supported and maintained by Microsoft helping reduce the amount of code to be maintained. Information about how to discover these is available in [Azure Verified Modules for Terraform](instructions/azure-verified-modules-terraform.instructions.md).

If an Azure Verified Module is not available for the resource, suggest creating one "in the style of" AVM in order to align to existing work and provide an opportunity to contribute upstream to the community.

Modules are appropriate for any resource set that will be used in multiple contexts to avoid duplicating code, and should not simply be basic wrappers for an existing resource.

Unless directed otherwise, offer to convert resources that are not using AVMs to equivalents, along with `moved()` blocks.

This promotes reuse, consistency, and helps improve the quality of the overall solution through re-using modules that have been independently tested.

An exception to this rule is if the user has been directed to use an internal private registry.

## 5. Follow recommended Terraform practices

- **Avoid explicit dependencies**: `depends_on` should only be used where a dependency is required that cannot be implicitly specified by referencing the resource output or module output. Necessary dependencies must be commented. Suggest removal of unnecessary dependencies. Never place an explicit dependency on a module output as it will cause inputs to a module to be unknown at plan time.

- **Iteration**: Use `count` for binary operations when creating 0 or 1 resources. Use `for_each` when more than 1 resource is needed. It can also be used for 0 resources if this helps maintain style consistency. Using an explicit iterator using a map ensures that resources can be added/removed without affecting others and provides stable resource addresses. Align with AVM specification TFNFR7 for consistent iteration patterns.

- **Data sources**: Data sources are appropriate in Terraform solutions (within the root folder), but should be avoided within modules. Prefer the use of module parameters that are explicit and avoid data source lookups inside the module. As an example, some modules that use AzAPI allow the parent_id to be specified, which is preferable to the fallback behaviour where the parent_id is inferred from the current subscription.

- **Parameterize**: Appropriate configurable variables for the solution. Remember that AVM exposes all variables and it is typically not worthwhile to repeat this in the solution code. Variables should be strongly typed with explicit `type` declarations (per TFNFR18), include comprehensive descriptions (per TFNFR17), and avoid nullable defaults for collection values (per TFNFR20). Default values aligning to Azure recommended practices should be used to reduce the number of inputs required.

- **Use outputs** to expose key resource attributes for other modules or user reference. Consider additional outputs beyond the minimum required (per TFFR2) that may be useful for solution consumers.

- Always target the latest stable Terraform version and Azure providers. In code, specify the required Terraform and provider versions to enforce this. Keep provider versions updated to get new features and fixes. Align with AVM specification TFFR3 for permitted provider versions.

## 6. Variable and Code Style Standards

Follow AVM-aligned coding standards in solution code to maintain consistency:

- **Variable naming**: Use snake_case for all variable names (per TFNFR4 and TFNFR16). Be descriptive and consistent with naming conventions.
- **Variable definitions**: All variables must have explicit type declarations (per TFNFR18) and comprehensive descriptions (per TFNFR17). Avoid nullable defaults for collection values (per TFNFR20) unless there's a specific need.
- **Sensitive variables**: Mark sensitive variables appropriately and avoid setting `sensitive = false` explicitly (per TFNFR22). Handle sensitive default values correctly (per TFNFR23).
- **Dynamic blocks**: Use dynamic blocks for optional nested objects where appropriate (per TFNFR12), and leverage `coalesce` or `try` functions for default values (per TFNFR13).
- **Code organization**: Consider using `locals.tf` specifically for local values (per TFNFR31) and ensure precise typing for locals (per TFNFR33).

## 7. Secrets

The best secret is one that does not need to be stored in code. Where possible, use mechanisms such as Managed Identities to avoid having to persist secrets.

Secrets that must be persisted should be stored in KeyVault unless directed to use another secret management solution.

If the module supports, use `ephemeral` secrets with write only parameters to avoid writing secrets to the terraform state file. At the time of writing, many modules do not support this, so consult module documentation and Hashicorp's documentation as appropriate to verify if this is available. Ephemeral secrets were introduced in Terraform CLI v1.11.

Never write secrets to the local file system as malware may use this to gain unauthorized access to systems.

Never add secrets to git.

Mark sensitive values accordingly to protect secrets. It is recommended to keep such inputs and outputs isolated from other attributes. Avoid outputting sensitive values unless there is a strong reason. Follow AVM specifications TFNFR19, TFNFR22, and TFNFR23 for handling sensitive data appropriately.

## 8. Folder Structure

People have mixed opinions about how to organize Terraform code, make initial suggestions and guide if requested, but do not make changes to the folder structure without asking the user.

A recommended starting point is:

- Ensure the root folder is the same across environments (dev, test, prod)
- Use tfvars to modify scale, sizing, and other environmental differences. In general, aim to keep environments similar. Cost is typically the driver for non-production environments.
- It is recommended to use explicitly named tfvars and then supply them (e.g. via `--var-file` or Terraform workspaces), rather than rely on auto tfvars, which may lead to unexpected or unintended consequences
- `tfvar` files are checked into source code and thus must not contain secret values.
- Consider placing the root module for the Terraform infrastructure in a dedicated folder, e.g. `./infra/` - this folder is suggested as Azure Developer CLI (AZD) expects this location, and use of AZD can help with team onboarding.
- Antipattern - branch per environment, repository per environment, folder per environment.

The choice of tooling may affect the guidance around folder structure. Unless told otherwise, assume you are working with the open source Terraform CLI without any additional tools such as Terragrunt or Terrateam.

### Example Structure

```text
my-azure-app/
├── infra/                          # Terraform root module (AZD compatible)
│   ├── main.tf                     # Core resources
│   ├── variables.tf                # Input variables
│   ├── outputs.tf                  # Outputs
│   ├── terraform.tf                # Provider configuration
│   ├── locals.tf                   # Local values
│   └── environments/               # Environment-specific configurations
│       ├── dev.tfvars              # Development environment
│       ├── test.tfvars             # Test environment
│       └── prod.tfvars             # Production environment
├── .github/workflows/              # CI/CD pipelines (if using github)
├── .azdo/                          # CI/CD pipelines (suggested if using github)
└── README.md                       # Documentation
```

**Example tfvars differentiation:**

- `environments/dev.tfvars`: Smaller SKUs, single region, minimal redundancy
- `environments/prod.tfvars`: Production SKUs, multi-region, high availability

**Usage with explicit tfvars:**

```bash
# Development deployment
terraform plan -var-file="environments/dev.tfvars"
terraform apply -var-file="environments/dev.tfvars"

# Production deployment
terraform plan -var-file="environments/prod.tfvars"
terraform apply -var-file="environments/prod.tfvars"
```

Modules should be sourced from AVM if available, or from a private registry if the user has been provided with one.  If these options are not available, custom local modules can be stored under `infra/modules`.

## 9. Testing and Validation

### Recommended Tools

- **tflint**: Lint Terraform code for best practices and provider-specific issues ([tflint docs](https://github.com/terraform-linters/tflint)).
- **terraform-docs**: Generate documentation for variables and outputs ([terraform-docs](https://terraform-docs.io/)).
- **tfsec**/**checkov**: Static security analysis of Terraform code ([tfsec](https://aquasecurity.github.io/tfsec/), [checkov](https://www.checkov.io/)).
- **pre-commit**: Automate formatting, linting, and validation with pre-commit hooks ([pre-commit](https://pre-commit.com/)).
- **terratest**: Automated integration testing for Terraform modules ([terratest](https://terratest.gruntwork.io/)).

**See also:** The [terraform-implement chatmode](chatmodes/terraform-implement.chatmode.md) for patterns on tool integration and workflow automation.

Follow testing best practices for Azure solutions:

- **Plan validation**: Always run `terraform plan` before applying changes to review the execution plan.
- **State management**: Understand state file implications and use remote state backends for team collaboration.
- **Environment separation**: Maintain separate state files and configurations for different environments.
- **Module testing**: When creating custom modules, implement appropriate testing strategies following AVM testing requirements where applicable.

## 10. Cost Management

- **Cost Validation Required**: Before suggesting expensive resources (Application Gateway, dedicated compute, premium storage tiers), confirm budget approval and necessity.
- **Environment-Appropriate Sizing**: Advise the user if deploying resources that are known to be expensive.
- **Cost Guardrails**: If no cost constraints are specified, ask for budget boundaries before proposing resource configurations.
- Assume that production environments favour resiliency over cost management, and non-production environments favour tighter cost controls. If existing code or the readme suggests otherwise, confirm the user's intentions. Make use of the guidance in `Folder Structure` to maintain this per-environment.

## 11. Azure-Specific Best Practices

### Resource Naming and Tagging

- Follow Azure naming conventions using the [Azure naming convention guidelines](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)
- Use consistent Azure region naming (per AVM specification TFNFR11) and consider using variables for locations to enable multi-region deployments
- Implement meaningful, consistent names that indicate the resource type, environment, and purpose (following AVM naming principles from TFNFR6 and TFNFR10)
- Implement consistent tagging strategy across all resources, for example:

  ```hcl
  locals {
    common_tags = {
      Environment   = var.environment
      Project       = var.project_name
      Owner         = var.owner
      CostCenter    = var.cost_center
      CreatedBy     = "terraform"
      CreatedDate   = formatdate("YYYY-MM-DD", timestamp())
    }
  }
  ```

- Ask the user what tags should be applied if this is not obvious. **Required tags must be confirmed before deployment**. Exceptions are possible if the solution is for a demo or throwaway environment, or if tagging is performed by an external component such as Azure Policy.

### Resource Group Strategy

- **Validate Resource Placement**: If target resource groups are specified, use them. Do not create new resource groups without confirmation.
- **Lifecycle Alignment**: Create resource groups strategically based on lifecycle and management boundaries only when not pre-defined
- Use descriptive resource group names that indicate purpose and environment

### Networking Considerations

- **Validate Network Boundaries**: When existing VNet/subnet IDs are provided, do not create conflicting network resources
- **Confirm Connectivity Requirements**: Before creating new VNets or modifying network configuration, confirm this aligns with existing network architecture
- Consider you may be deploying into an application landing zone which has been set up appropriately with vnet resources, this will likely be indicated by parameters supplying an existing vnet or subnet resource ID
- Use Network Security Groups (NSGs) and Application Security Groups (ASGs) appropriately in your Terraform code
- For PaaS services, implement private endpoints in Terraform when required, otherwise configure resource firewall restrictions

### Security and Compliance

- Use Managed Identities wherever possible instead of service principals in Terraform resources
- Implement Key Vault references for secrets management with appropriate access policies
- Enable diagnostic settings for audit trails using Terraform diagnostic setting resources

## 12. Minimal Dependencies

- **Confirmation Required**: Do not introduce additional providers or modules beyond the project's scope without explicit user confirmation
- **Justify Dependencies**: If a special provider (e.g., `random`, `tls`) or external module is needed:
  - Add a comment explaining the necessity
  - Ensure the user approves it
  - Verify it doesn't conflict with organizational policy
- Keep the infrastructure stack lean and avoid unnecessary complexity

## 13. Ensure Idempotency

- Write configurations that can be applied repeatedly with the same outcome
- **Avoid non-idempotent actions**:
  - Scripts that run on every apply
  - Resources that might conflict if created twice
- **Test by doing multiple `terraform apply` runs** and ensure the second run results in zero changes
- Use resource lifecycle settings or conditional expressions to handle drift or external changes gracefully

## 14. State Management

- **Use a remote backend** (like Azure Storage with state locking) to store Terraform state securely
- Enable team collaboration
- **Never commit state files** to source control
- This prevents conflicts and keeps the infrastructure state consistent
- Configure backend with encryption at rest and in transit

## 15. CI/CD Integration

### Azure DevOps Integration

This section applies if the user is using Azure DevOps.

- Use Azure DevOps pipelines for Terraform automation
- Implement plan/apply workflow with manual approval gates
- Store Terraform state in Azure Storage Account with versioning enabled
- Use service connections with appropriate RBAC permissions

### GitHub Actions Integration

This section applies if the user is using GitHub (there is a .github folder in the repository).

- Use GitHub Actions with OIDC for secure authentication to Azure
- Implement branch protection rules requiring successful Terraform plans
- Store sensitive variables in GitHub Secrets or Azure Key Vault

### Pipeline Best Practices

- Always run `terraform plan` in pull requests
- Use `terraform validate` and `terraform fmt -check` in CI
- Implement policy as code validation (Azure Policy, OPA, Sentinel)
- Run linting checks, such as tflint, terraform-docs
- Run security CLI tools to analyse terraform code and terraform plans
- Consider CLI tools to keep costs visible to engineering teams.
- Consider matrix builds for multi-environment deployments

## 16. Testing Strategies

### Static Analysis

- Use `terraform validate` for syntax validation
- Implement `tflint` for advanced linting
- Use `terratest` for automated integration testing
- Run security scanning with tools like `checkov` or `tfsec`

### Integration Testing

- Test infrastructure provisioning in isolated environments
- Validate connectivity and functionality post-deployment
- Use blue-green deployments for production changes

## 17. Documentation

### Documentation Automation Tools

- **terraform-docs**: Auto-generate markdown docs for variables and outputs.
- **pre-commit**: Enforce documentation updates and formatting.

**Reference:** For module documentation standards, see [Azure Verified Modules for Terraform](instructions/azure-verified-modules-terraform.instructions.md).

- **Maintain up-to-date documentation**
- **Update README.md** with any new variables, outputs, or usage instructions whenever the code changes
- Consider using tools like `terraform-docs` for automation
- **Update architecture diagrams** to reflect infrastructure changes after each significant update
- Well-documented code and diagrams ensure the whole team understands the infrastructure
- Include example `terraform.tfvars` files with explanations
- **Variable descriptions**: Provide meaningful descriptions for all variables that explain their purpose and expected values (per TFNFR17)
- **Output descriptions**: Document all outputs with clear descriptions of what they represent (per TFNFR17)

## Pre-deployment Validation

- **Pre-deployment Validation Required**: Run `terraform validate` and review the `terraform plan` output before applying changes
- **Architectural Compliance Check**: Before applying changes, verify:
  - Resource placement aligns with specified constraints
  - Network changes don't bypass security boundaries  
  - Resource sizing stays within cost parameters
  - Integration points are validated
- Recommend adding Terraform-related pre-commit hooks if they are not present
- Catch errors or unintended modifications early
- **Consider implementing automated checks**:
  - CI pipeline
  - Pre-commit hooks
  - Enforce formatting, linting, and basic validation
- Use `terraform show` to understand current state
- Implement drift detection with regular `terraform plan` runs
