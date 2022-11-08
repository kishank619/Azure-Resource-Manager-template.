{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "adminUsername": {
      "type": "string",
      "metadata": {
        "description": "Admin username"
      }
    },
    "adminPassword": {
      "type": "securestring",
      "metadata": {
        "description": "Admin password"
      }
    },
    "vmNamePrefix": {
      "type": "string",
      "defaultValue": "az104-10-vm",
      "metadata": {
        "description": "VM name prefix"
      }
    },
    "pipNamePrefix": {
      "type": "string",
      "defaultValue": "az104-10-pip",
      "metadata": {
        "description": "Public IP address name prefix"
      }
    },
    "nicNamePrefix": {
      "type": "string",
      "defaultValue": "az104-10-nic",
      "metadata": {
        "description": "Nic name prefix"
      }
    },
    "imagePublisher": {
      "type": "string",
      "defaultValue": "MicrosoftWindowsServer",
      "metadata": {
        "description": "Image Publisher"
      }
    },
    "imageOffer": {
      "type": "string",
      "defaultValue": "WindowsServer",
      "metadata": {
        "description": "Image Offer"
      }
    },
    "imageSKU": {
      "type": "string",
      "defaultValue": "2019-Datacenter",
      "allowedValues": [
        "2019-Datacenter",
        "2019-Datacenter-Server-Core",
        "2019-Datacenter-Server-Core-smalldisk"
      ],
      "metadata": {
        "description": "Image SKU"
      }
    },
    "vmSize": {
      "type": "string",
      "defaultValue": "Standard_D2s_v3",
      "metadata": {
        "description": "VM size"
      }
    },
    "virtualNetworkName": {
      "type": "string",
      "defaultValue": "az104-10-vnet",
      "metadata": {
        "description": "Virtual network name"
      }
    },
    "addressPrefix": {
      "type": "string",
      "defaultValue": "10.0.0.0/24",
      "metadata": {
        "description": "Virtual network address prefix"
      }
    },
    "virtualNetworkResourceGroup": {
      "type": "string",
      "defaultValue": "az104-10-rg0",
      "metadata": {
        "description": "Resource group of the VNet"
      }
    },
    "subnet0Name": {
      "type": "string",
      "defaultValue": "subnet0",
      "metadata": {
        "description": "VNet first subnet name"
      }
    },
    "subnet0Prefix": {
      "type": "string",
      "defaultValue": "10.0.0.0/26",
      "metadata": {
        "description": "VNet first subnet prefix"
      }
    },
    "nsgName": {
      "type": "string",
      "defaultValue": "az104-10-nsg01",
      "metadata": {
        "description": "Network security group name"
      }
    }
  },
  "variables": {
    "vnetID": "[resourceId(parameters('virtualNetworkResourceGroup'), 'Microsoft.Network/virtualNetworks', parameters('virtualNetworkName'))]",
    "subnetRef": "[resourceId('Microsoft.Network/virtualNetworks/subnets', parameters('virtualNetworkName'), parameters('subnet0Name'))]",
    "numberOfInstances": 2,
    "computeAPIVersion": "2018-10-01",
    "networkAPIVersion": "2018-12-01"
  },
  "resources": [
    {
      "type": "Microsoft.Network/networkInterfaces",
      "name": "[concat(parameters('nicNamePrefix'), copyindex())]",
      "apiVersion": "[variables('networkAPIVersion')]",
      "location": "[resourceGroup().location]",
      "copy": {
        "name": "nicLoop",
        "count": "[variables('numberOfInstances')]"
      },
      "dependsOn": [
        "[resourceId('Microsoft.Network/virtualNetworks/',parameters('virtualNetworkName'))]",
        "[resourceId('Microsoft.Network/networkSecurityGroups/',parameters('nsgName'))]",
        "pipLoop"
      ],
      "properties": {
        "ipConfigurations": [
          {
            "name": "ipconfig1",
            "properties": {
              "privateIPAllocationMethod": "Dynamic",
              "subnet": {
                "id": "[variables('subnetRef')]"
              },
              "publicIpAddress": {
                "id": "[resourceId('Microsoft.Network/publicIpAddresses',concat(parameters('pipNamePrefix'),copyindex()))]"
              }
            }
          }
        ],
        "networkSecurityGroup": {
          "id": "[resourceId('Microsoft.Network/networkSecurityGroups', parameters('nsgName'))]"
        }
      }
    },
    {
      "type": "Microsoft.Network/virtualNetworks",
      "name": "[parameters('virtualNetworkName')]",
      "apiVersion": "[variables('networkAPIVersion')]",
      "location": "[resourceGroup().location]",
      "properties": {
        "addressSpace": {
          "addressPrefixes": [
            "[parameters('addressPrefix')]"
          ]
        },
        "subnets": [
          {
            "name": "[parameters('subnet0Name')]",
            "properties": {
              "addressPrefix": "[parameters('subnet0Prefix')]"
            }
          }
        ]
      }
    },
    {
      "type": "Microsoft.Network/publicIpAddresses",
      "name": "[concat(parameters('pipNamePrefix'), copyindex())]",
      "apiVersion": "[variables('networkApiVersion')]",
      "copy": {
        "name": "pipLoop",
        "count": "[variables('numberOfInstances')]"
      },
      "location": "[resourceGroup().location]",
      "properties": {
        "publicIpAllocationMethod": "Dynamic"
      }
    },
    {
      "type": "Microsoft.Network/networkSecurityGroups",
      "name": "[parameters('nsgName')]",
      "apiVersion": "[variables('networkApiVersion')]",
      "location": "[resourceGroup().location]",
      "properties": {
        "securityRules": [
          {
            "name": "default-allow-rdp",
            "properties": {
              "priority": 1000,
              "sourceAddressPrefix": "*",
              "protocol": "Tcp",
              "destinationPortRange": "3389",
              "access": "Allow",
              "direction": "Inbound",
              "sourcePortRange": "*",
              "destinationAddressPrefix": "*"
            }
          }
        ]
      }
    },
    {
      "type": "Microsoft.Compute/virtualMachines",
      "name": "[concat(parameters('vmNamePrefix'), copyindex())]",
      "apiVersion": "[variables('computeAPIVersion')]",
      "copy": {
        "name": "virtualMachineLoop",
        "count": "[variables('numberOfInstances')]"
      },
      "location": "[resourceGroup().location]",
      "dependsOn": [
        "nicLoop"
      ],
      "properties": {
        "hardwareProfile": {
          "vmSize": "[parameters('vmSize')]"
        },
        "osProfile": {
          "computerName": "[concat(parameters('vmNamePrefix'), copyIndex())]",
          "adminUsername": "[parameters('adminUsername')]",
          "adminPassword": "[parameters('adminPassword')]"
        },
        "storageProfile": {
          "imageReference": {
            "publisher": "[parameters('imagePublisher')]",
            "offer": "[parameters('imageOffer')]",
            "sku": "[parameters('imageSKU')]",
            "version": "latest"
          },
          "osDisk": {
            "createOption": "FromImage",
            "managedDisk": {
               "storageAccountType": "Standard_LRS"
            }
          }
        },
        "networkProfile": {
          "networkInterfaces": [
            {
              "id": "[resourceId('Microsoft.Network/networkInterfaces',concat(parameters('nicNamePrefix'),copyindex()))]"
            }
          ]
        }
      }
    },
    {
      "type": "Microsoft.Compute/virtualMachines/extensions",
        "name": "[concat(parameters('vmNamePrefix'), copyindex(), '/customScriptExtension')]",
        "apiVersion": "2018-06-01",
        "location": "[resourceGroup().location]",
        "copy": {
          "name": "cSELoop",
          "count": "[variables('numberOfInstances')]"
        },
        "dependsOn": [
          "[concat(parameters('vmNamePrefix'), copyindex())]"
        ],
        "properties": {
          "publisher": "Microsoft.Compute",
          "type": "CustomScriptExtension",
          "typeHandlerVersion": "1.10",
          "autoUpgradeMinorVersion": true,
          "Settings": {
            "commandToExecute": "powershell.exe Set-ExecutionPolicy Bypass -Scope Process -Force && powershell.exe Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1')) && powershell.exe c:\\programdata\\chocolatey\\choco.exe install microsoft-edge -y"
         }
        }
    }
  ]
}
