{
  "vmPolicy": {
    "mode": "All",
    "policyRule": {
      "if": {
        "allOf": [
          {
            "field": "type",
            "equals": "Microsoft.Compute/virtualMachines"
          },
          {
            "field": "Microsoft.Compute/virtualMachines/sku.name",
            "notIn": ["Standard_B1s"]
          }
        ]
      },
      "then": {
        "effect": "Deny"
      }
    }
  },
  "storagePolicy": {
    "mode": "All",
    "policyRule": {
      "if": {
        "allOf": [
          {
            "field": "type",
            "equals": "Microsoft.Storage/storageAccounts"
          },
          {
            "field": "Microsoft.Storage/storageAccounts/sku.name",
            "notIn": ["Standard_LRS"]
          }
        ]
      },
      "then": {
        "effect": "Deny"
      }
    }
  }
}