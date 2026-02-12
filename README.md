# Azure Policy — Tagging Governance Initiative

Implement enterprise-ready tagging governance in Azure using Azure Policy. This repo provides production-hardened policy definitions that require specific tags on all resources and validate their values against approved lists.

## The Problem

Without tag enforcement, Azure environments quickly become ungovernable:

- Resources deployed with no ownership or cost attribution
- Inconsistent tag names and values across teams
- Compliance reporting becomes unreliable
- FinOps cost allocation breaks down

Consistent tagging is critical for [Azure Cost Management](https://learn.microsoft.com/azure/cost-management-billing/costs/quick-acm-cost-analysis) exports and FinOps reporting. Without it, cost attribution is guesswork.

## Architecture: Initiative-Based Approach

Rather than a single monolithic policy, this repo uses the **enterprise-recommended pattern** of splitting concerns into separate policy definitions grouped in an Initiative (Policy Set):

| Policy | Responsibility | Effect |
|--------|---------------|--------|
| **Require Tag Exists** | Ensures `Environment`, `Owner`, and `CostCenter` tags are present | Audit / Deny |
| **Enforce Allowed Values** | Validates tag values against approved lists | Audit / Deny |
| **Inherit Tags from Resource Group** | Auto-applies missing tags from the parent Resource Group | Modify |

This gives you:
- Better compliance reporting (know *which* check failed)
- Easier exemptions per policy
- Reusability across subscriptions and management groups

> This initiative is designed to be assigned at the **management group level** to ensure consistent tagging across all subscriptions.

---

## Policy Definitions

### Policy 1: Require Tags Exist

Ensures all three mandatory tags are present on every resource.

> **Important:** `mode: "Indexed"` is required for tag evaluation because tags are only available on indexed resource types. Changing this to `All` will break tag enforcement.

```json
{
  "mode": "Indexed",
  "metadata": {
    "category": "Tags",
    "version": "1.0.0"
  },
  "policyRule": {
    "if": {
      "anyOf": [
        {
          "field": "tags['Environment']",
          "exists": "false"
        },
        {
          "field": "tags['Owner']",
          "exists": "false"
        },
        {
          "field": "tags['CostCenter']",
          "exists": "false"
        }
      ]
    },
    "then": {
      "effect": "[parameters('effect')]"
    }
  },
  "parameters": {
    "effect": {
      "type": "String",
      "allowedValues": [
        "Audit",
        "Deny",
        "Disabled"
      ],
      "defaultValue": "Audit",
      "metadata": {
        "displayName": "Policy Effect",
        "description": "Start with Audit, switch to Deny after remediation"
      }
    }
  }
}
```

### Policy 2: Enforce Allowed Tag Values

Validates that tag values match approved lists. Uses `toLower()` to handle case-sensitivity.

```json
{
  "mode": "Indexed",
  "metadata": {
    "category": "Tags",
    "version": "1.0.0"
  },
  "policyRule": {
    "if": {
      "anyOf": [
        {
          "allOf": [
            {
              "field": "tags['Environment']",
              "exists": "true"
            },
            {
              "value": "[toLower(field('tags[''Environment'']'))]",
              "notIn": "[parameters('allowedEnvironment')]"
            }
          ]
        },
        {
          "allOf": [
            {
              "field": "tags['CostCenter']",
              "exists": "true"
            },
            {
              "value": "[toLower(field('tags[''CostCenter'']'))]",
              "notIn": "[parameters('allowedCostCenter')]"
            }
          ]
        }
      ]
    },
    "then": {
      "effect": "[parameters('effect')]"
    }
  },
  "parameters": {
    "effect": {
      "type": "String",
      "allowedValues": [
        "Audit",
        "Deny",
        "Disabled"
      ],
      "defaultValue": "Audit",
      "metadata": {
        "displayName": "Policy Effect",
        "description": "Start with Audit, switch to Deny after remediation"
      }
    },
    "allowedEnvironment": {
      "type": "Array",
      "defaultValue": [
        "dev",
        "test",
        "staging",
        "production"
      ],
      "metadata": {
        "displayName": "Allowed Environment values",
        "description": "Valid lowercase values for the Environment tag"
      }
    },
    "allowedCostCenter": {
      "type": "Array",
      "defaultValue": [
        "it",
        "finance",
        "engineering",
        "marketing",
        "operations"
      ],
      "metadata": {
        "displayName": "Allowed CostCenter values",
        "description": "Valid lowercase values for the CostCenter tag"
      }
    }
  }
}
```

> **Note:** The `Owner` tag is intentionally excluded from value enforcement. Owner lists change frequently and are better validated through naming conventions or integration with Entra ID.

### Policy 3: Inherit Tags from Resource Group (Modify)

Auto-applies tags from the parent Resource Group when resources are missing them. This is critical because **Azure does NOT automatically inherit tags from Resource Groups** — a common misconception.

Use the built-in policy definition `Inherit a tag from the resource group if missing` for this. This is a single policy definition — you create three separate **assignments**, each with a different `tagName` parameter value:

| Assignment | `tagName` Parameter | Policy Definition ID |
|------------|---------------------|---------------------|
| Inherit `Environment` tag from RG | `Environment` | `ea3f2387-9b95-492a-a190-fcbfef9b1aac` |
| Inherit `Owner` tag from RG | `Owner` | `ea3f2387-9b95-492a-a190-fcbfef9b1aac` |
| Inherit `CostCenter` tag from RG | `CostCenter` | `ea3f2387-9b95-492a-a190-fcbfef9b1aac` |

---

## Deployment

### Quick Start

**Portal** (recommended): Create the two custom policy definitions → create the Tagging Governance Initiative → assign the initiative with Effect = `Audit` at the management group (recommended) or subscription level.

**Automation**: Use the Azure CLI or Bicep examples below.

### Safe Rollout Pattern

```
Phase 1: Audit     →  Assign with effect = "Audit"
                       Monitor compliance for 2-4 weeks
                       Identify non-compliant resources

Phase 2: Remediate →  Fix existing resources via remediation tasks
                       Update IaC pipelines
                       Educate teams

Phase 3: Deny      →  Switch effect to "Deny"
                       Only after >95% compliance achieved
```

### Deploy via Azure Portal (Recommended for Getting Started)

#### Step 1 — Create the Policy Definitions

1. Open the [Azure Portal](https://portal.azure.com)
2. Navigate to **Policy** → **Definitions**
3. Click **+ Policy definition**
4. Set:
   - **Definition location**: Select your subscription or management group
   - **Name**: `Require Environment, Owner, and CostCenter tags`
   - **Category**: Click **Use existing** → select **Tags**
5. Paste the JSON from **Policy 1: Require Tags Exist** (above) into the **Policy rule** field
6. Click **Save**
7. Repeat for **Policy 2: Enforce Allowed Tag Values**

#### Step 2 — Create the Initiative (Policy Set)

1. Go to **Policy** → **Definitions**
2. Click **+ Initiative definition**
3. Set:
   - **Name**: `Tagging Governance Initiative`
   - **Category**: **Tags**
4. Click **Add policy definition(s)** and add:
   - `Require Environment, Owner, and CostCenter tags`
   - `Enforce Allowed Tag Values`
   - Built-in: `Inherit a tag from the resource group` (add this 3 times — once for each tag: `Environment`, `Owner`, `CostCenter`)
5. Configure parameter values on the **Parameters** tab
6. Click **Save**

#### Step 3 — Assign the Initiative (Audit Mode)

1. Go to **Policy** → **Assignments**
2. Click **Assign initiative**
3. Select your **Tagging Governance Initiative**
4. **Scope**: Select your subscription (or management group)
5. On the **Parameters** tab:
   - Set **Effect** = `Audit`
   - Set allowed Environment values (e.g., `dev`, `test`, `staging`, `production`)
   - Set allowed CostCenter values (e.g., `it`, `finance`, `engineering`)
6. Click **Review + create** → **Create**

#### Step 4 — Monitor and Switch to Deny

1. Go to **Policy** → **Compliance**
2. Monitor your initiative's compliance percentage over 2–4 weeks
3. Remediate non-compliant resources
4. Once compliance exceeds 95%, edit the assignment and change **Effect** to `Deny`

### Deploy via Azure CLI (Automation)

**Create the policy definition:**

```bash
az policy definition create \
  --name "require-tags-exist" \
  --display-name "Require Environment, Owner, and CostCenter tags" \
  --description "Ensures all resources have the required governance tags" \
  --rules ./policies/require-tags-exist.json \
  --params ./policies/require-tags-exist.params.json \
  --mode Indexed
```

**Assign with Audit effect:**

```bash
az policy assignment create \
  --name "require-tags-audit" \
  --display-name "Require Tags (Audit)" \
  --policy "require-tags-exist" \
  --params '{"effect": {"value": "Audit"}}' \
  --scope "/subscriptions/<subscription-id>"
```

**Switch to Deny when ready:**

```bash
az policy assignment update \
  --name "require-tags-audit" \
  --set "parameters.effect.value=Deny"
```

### Deploy via Bicep (Infrastructure as Code)

```bicep
targetScope = 'subscription'

resource policyDef 'Microsoft.Authorization/policyDefinitions@2021-06-01' = {
  name: 'require-tags-exist'
  properties: {
    displayName: 'Require Environment, Owner, and CostCenter tags'
    policyType: 'Custom'
    mode: 'Indexed'
    metadata: {
      category: 'Tags'
      version: '1.0.0'
    }
    parameters: {
      effect: {
        type: 'String'
        allowedValues: [
          'Audit'
          'Deny'
          'Disabled'
        ]
        defaultValue: 'Audit'
        metadata: {
          displayName: 'Policy Effect'
        }
      }
    }
    policyRule: {
      if: {
        anyOf: [
          { field: 'tags[\'Environment\']', exists: 'false' }
          { field: 'tags[\'Owner\']', exists: 'false' }
          { field: 'tags[\'CostCenter\']', exists: 'false' }
        ]
      }
      then: {
        effect: '[parameters(\'effect\')]'
      }
    }
  }
}

resource policyAssignment 'Microsoft.Authorization/policyAssignments@2021-06-01' = {
  name: 'require-tags-audit'
  properties: {
    displayName: 'Require Tags (Audit Mode)'
    policyDefinitionId: policyDef.id
    parameters: {
      effect: { value: 'Audit' }
    }
  }
}
```

---

## Key Lessons Learned

1. **Never deploy Deny without Audit first** — you will break pipelines, Marketplace deployments, and Microsoft-managed resources.

2. **Tag comparisons are case-sensitive** — `toLower()` normalization is essential in production.

3. **Resources do NOT inherit tags from Resource Groups** — you need a companion `Modify` policy.

4. **Split policies beat monolithic ones** — easier compliance reporting, exemptions, and reuse.

5. **Some resources can't be tagged** — non-taggable resource types (role assignments, policy assignments, etc.) are automatically skipped.

6. **System resources need consideration** — AKS managed resources, Defender artifacts, and Marketplace solutions may need assignment-level exclusions via `notScopes` or exemptions.

---

## Exemptions vs Exclusions

Use **policy exemptions** for temporary or approved exceptions rather than excluding scopes with `notScopes`.

- Provides an **audit trail** of who approved the exception and why
- Supports **expiration dates** so exemptions don't linger forever
- Enables **justification and approval workflows** through integration with governance processes

Reserve `notScopes` for permanent exclusions only, such as sandbox subscriptions or dedicated lab environments.

---

## Useful References

- [Azure Policy overview](https://learn.microsoft.com/azure/governance/policy/overview)
- [Azure Policy tag tutorials](https://learn.microsoft.com/azure/governance/policy/tutorials/govern-tags)
- [Built-in tag policies](https://learn.microsoft.com/azure/governance/policy/samples/built-in-policies#tags)
- [Cloud Adoption Framework — Tagging strategy](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-tagging)
- [Policy definition structure](https://learn.microsoft.com/azure/governance/policy/concepts/definition-structure)

---

## License

MIT
