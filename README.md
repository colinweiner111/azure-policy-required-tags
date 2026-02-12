# Azure Policy — Required Tags with Allowed Values

Enforce mandatory tagging standards across your Azure environment using Azure Policy. This repo provides production-hardened policy definitions that require specific tags on all resources and validate their values against approved lists.

## The Problem

Without tag enforcement, Azure environments quickly become ungovernable:

- Resources deployed with no ownership or cost attribution
- Inconsistent tag names and values across teams
- Compliance reporting becomes unreliable
- FinOps cost allocation breaks down

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

---

## Policy Definitions

### Policy 1: Require Tags Exist

Ensures all three mandatory tags are present on every resource.

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

Use the built-in policy definitions for this:

| Built-in Policy | Policy Definition ID |
|-----------------|---------------------|
| Inherit `Environment` tag from RG | `ea3f2387-9b95-492a-a190-fcbfef9b1aac` |
| Inherit `Owner` tag from RG | `ea3f2387-9b95-492a-a190-fcbfef9b1aac` |
| Inherit `CostCenter` tag from RG | `ea3f2387-9b95-492a-a190-fcbfef9b1aac` |

> The same built-in definition is reused with different `tagName` parameter values per assignment.

---

## Original Single-Policy Version

For simpler environments, this was the starting point — a single policy that combines existence checks and value validation:

```json
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "anyOf": [
        {
          "field": "tags['Environment']",
          "exists": "false"
        },
        {
          "field": "tags['Environment']",
          "notIn": "[parameters('allowedEnvironment')]"
        },
        {
          "field": "tags['Owner']",
          "exists": "false"
        },
        {
          "field": "tags['Owner']",
          "notIn": "[parameters('allowedOwner')]"
        },
        {
          "field": "tags['CostCenter']",
          "exists": "false"
        },
        {
          "field": "tags['CostCenter']",
          "notIn": "[parameters('allowedCostCenter')]"
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {
    "allowedEnvironment": {
      "type": "Array",
      "metadata": {
        "displayName": "Allowed Environment values",
        "description": "Valid values for the 'Environment' tag"
      }
    },
    "allowedOwner": {
      "type": "Array",
      "metadata": {
        "displayName": "Allowed Owner values",
        "description": "Valid values for the 'Owner' tag"
      }
    },
    "allowedCostCenter": {
      "type": "Array",
      "metadata": {
        "displayName": "Allowed CostCenter values",
        "description": "Valid values for the 'CostCenter' tag"
      }
    }
  }
}
```

### Why We Improved It

| Issue | Risk | Fix Applied |
|-------|------|-------------|
| Hard-coded `deny` effect | Blocks all deployments immediately on assignment | Parameterized effect with `Audit` default |
| Case-sensitive comparisons | `prod` ≠ `Prod` ≠ `PROD` — blocks valid deployments | `toLower()` normalization |
| No `defaultValue` on parameters | Assignment fails if values not provided | Added sensible defaults |
| Single monolithic policy | Hard to troubleshoot, exempt, and reuse | Split into Initiative |
| No tag inheritance | Devs assume RG tags flow to resources (they don't) | Added Modify companion policy |
| Owner value enforcement | Owner lists change constantly | Require existence only, skip value check |
| No `metadata.category` | Policy doesn't appear correctly in Portal grouping | Added `"category": "Tags"` |

---

## Deployment

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

### Deploy via Azure CLI

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

**Assign with Audit effect (Phase 1):**

```bash
az policy assignment create \
  --name "require-tags-audit" \
  --display-name "Require Tags (Audit)" \
  --policy "require-tags-exist" \
  --params '{"effect": {"value": "Audit"}}' \
  --scope "/subscriptions/<subscription-id>"
```

**Switch to Deny (Phase 3):**

```bash
az policy assignment update \
  --name "require-tags-audit" \
  --set "parameters.effect.value=Deny"
```

### Deploy via Bicep

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

5. **Some resources can't be tagged** — `mode: "Indexed"` correctly skips non-taggable resource types (role assignments, policy assignments, etc.).

6. **System resources need consideration** — AKS managed resources, Defender artifacts, and Marketplace solutions may need assignment-level exclusions via `notScopes` or exemptions.

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
