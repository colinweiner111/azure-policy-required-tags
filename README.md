# Azure Tagging Governance — Portal Quick Start

Implement simple, production-ready tagging governance in Azure using Azure Policy — no CLI or IaC required.

## What You Will Accomplish

In about 15 minutes, you will deploy Azure Policy that:

- **Requires** `Environment`, `Owner`, and `CostCenter` tags on all resources
- **Automatically inherits** tags from Resource Groups
- **Starts in Audit mode** so nothing breaks
- **Prepares** your environment for Cost Management and FinOps reporting

This is a safe, recommended starting point for new Azure environments.

---

## Deploy in the Azure Portal

Follow these steps in the [Azure Portal](https://portal.azure.com). You will create two custom policies, combine them into an initiative, and assign it in Audit mode.

### Step 1 — Create the "Require Tags" Policy

1. Open the **Azure Portal**
2. Search for **Policy**
3. Go to **Policy** → **Definitions**
4. Click **+ Policy definition**
5. Fill in:

   | Setting | Value |
   |---------|-------|
   | Definition location | Your subscription or management group |
   | Name | `Require Environment, Owner, and CostCenter tags` |
   | Category | Tags |

6. In **Policy rule**, paste the JSON from [Policy 1](#policy-1-require-tags-exist) below
7. Click **Save**

### Step 2 — Create the "Allowed Values" Policy

Repeat the same process:

1. Click **+ Policy definition**
2. Fill in:

   | Setting | Value |
   |---------|-------|
   | Name | `Enforce Allowed Tag Values` |
   | Category | Tags |

3. Paste the JSON from [Policy 2](#policy-2-enforce-allowed-tag-values) below
4. Click **Save**

You now have the two custom policies ready.

### Step 3 — Create the Tagging Governance Initiative

Best practice is to group related policies into an initiative so they can be assigned and managed together.

1. Go to **Policy** → **Definitions**
2. Click **+ Initiative definition**
3. Fill in:

   | Setting | Value |
   |---------|-------|
   | Initiative location | Your subscription or management group |
   | Name | `Tagging Governance Initiative` |
   | Category | Tags |

4. Click **Add policy definition(s)** and add:
   - `Require Environment, Owner, and CostCenter tags`
   - `Enforce Allowed Tag Values`
   - Built-in: `Inherit a tag from the resource group if missing` — add this **three times**:
     - Once for `Environment`
     - Once for `Owner`
     - Once for `CostCenter`
5. Go to the **Initiative parameters** tab and create the following parameters (these surface on the assignment so you can change them later). Click **…** on each to set the default value — array parameters **must** have an array default or you'll get a type-mismatch error:

   | Name | Display name | Type | Default Value |
   |------|-------------|------|---------------|
   | `effect` | Effect | String | `Audit` |
   | `allowedEnvironments` | Allowed Environment values | Array | `["dev","test","staging","production"]` |
   | `allowedCostCenters` | Allowed CostCenter values | Array | `["it","finance","engineering","marketing","operations"]` |

6. Go to the **Policy parameters** tab. **Uncheck** "Only show parameters that need input or review" to see all rows. Then set each row:

   | Reference ID | Parameter name | Value Type | Value(s) |
   |-------------|---------------|------------|----------|
   | Require Environment, Owner, and CostCenter tags_1 | Policy Effect | Use Initiative Parameter | `effect` |
   | Enforce Allowed Tag Values_1 | Policy Effect | Use Initiative Parameter | `effect` |
   | Enforce Allowed Tag Values_1 | Allowed CostCenter values | Use Initiative Parameter | `allowedCostCenters` |
   | Enforce Allowed Tag Values_1 | Allowed Environment values | Use Initiative Parameter | `allowedEnvironments` |
   | Inherit a tag from the resource group if missing_1 | Tag Name | Set value | type `Environment` |
   | Inherit a tag from the resource group if missing_2 | Tag Name | Set value | type `Owner` |
   | Inherit a tag from the resource group if missing_3 | Tag Name | Set value | type `CostCenter` |

7. Click **Review + create**, then **Create**

Your initiative is ready.

### Step 4 — Assign the Initiative (Start in Audit Mode)

1. Go to **Policy** → **Assignments**
2. Click **Assign initiative**
3. On the **Basics** tab:
   - **Scope** — pick your Management Group (recommended) or Subscription
   - **Initiative definition** — click **…** and select **Tagging Governance Initiative**
   - **Assignment name** auto-fills; leave it or rename
   - **Policy enforcement** — leave **Enabled**
4. Go to the **Parameters** tab:
   - Set **Effect** → `Audit`
   - Set **Environment allowed values** → `dev`, `test`, `staging`, `production`
   - Set **CostCenter allowed values** → your organization's values
5. Click **Review + create** → **Create**

Tagging governance is now deployed.

### Step 5 — Monitor and Switch to Deny Later

1. Go to **Policy** → **Compliance**
2. Monitor compliance for 2–4 weeks
3. Remediate non-compliant resources
4. When compliance reaches ~95%, edit the assignment and change **Effect** → `Deny`

New resources must now follow tagging standards.

---

## What You Just Deployed

This solution includes three governance controls:

| Control | Purpose |
|---------|---------|
| **Require tags exist** | Ensures every resource has core tags |
| **Enforce allowed values** | Standardizes tag values |
| **Inherit tags from RG** | Automatically applies missing tags |

Together, these provide a strong tagging foundation without breaking deployments.

---

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

This policy uses parameters to define the allowed tag values and enforces them. It uses `toLower()` to handle case-sensitivity.

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

## CLI and Bicep Automation

For teams ready to automate, these examples provide the same deployment via code.

### Azure CLI

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

### Bicep

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
