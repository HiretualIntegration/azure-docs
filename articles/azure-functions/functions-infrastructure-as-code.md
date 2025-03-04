---
title: Automate function app resource deployment to Azure
description: Learn how to build an Azure Resource Manager template that deploys your function app.

ms.assetid: d20743e3-aab6-442c-a836-9bcea09bfd32
ms.topic: conceptual
ms.date: 04/03/2019
ms.custom: fasttrack-edit, devx-track-azurepowershell
---

# Automate resource deployment for your function app in Azure Functions

You can use an Azure Resource Manager template to deploy a function app. This article outlines the required resources and parameters for doing so. You might need to deploy other resources, depending on the [triggers and bindings](functions-triggers-bindings.md) in your function app.

For more information about creating templates, see [Authoring Azure Resource Manager templates](../azure-resource-manager/templates/syntax.md).

For sample templates, see:
- [Function app on Consumption plan]
- [Function app on Azure App Service plan]

## Required resources

An Azure Functions deployment typically consists of these resources:

| Resource                                                                           | Requirement | Syntax and properties reference                                                         |
|------------------------------------------------------------------------------------|-------------|-----------------------------------------------------------------------------------------|
| A function app                                                                     | Required    | [Microsoft.Web/sites](/azure/templates/microsoft.web/sites)                             |
| An [Azure Storage](../storage/index.yml) account                                   | Required    | [Microsoft.Storage/storageAccounts](/azure/templates/microsoft.storage/storageaccounts) |
| An [Application Insights](../azure-monitor/app/app-insights-overview.md) component | Optional    | [Microsoft.Insights/components](/azure/templates/microsoft.insights/components)         |
| A [hosting plan](./functions-scale.md)                                             | Optional<sup>1</sup>    | [Microsoft.Web/serverfarms](/azure/templates/microsoft.web/serverfarms)                 |

<sup>1</sup>A hosting plan is only required when you choose to run your function app on a [Premium plan](./functions-premium-plan.md) or on an [App Service plan](../app-service/overview-hosting-plans.md).

> [!TIP]
> While not required, it is strongly recommended that you configure Application Insights for your app.

<a name="storage"></a>
### Storage account

An Azure storage account is required for a function app. You need a general purpose account that supports blobs, tables, queues, and files. For more information, see [Azure Functions storage account requirements](storage-considerations.md#storage-account-requirements).

```json
{
    "type": "Microsoft.Storage/storageAccounts",
    "name": "[variables('storageAccountName')]",
    "apiVersion": "2019-06-01",
    "location": "[resourceGroup().location]",
    "kind": "StorageV2",
    "sku": {
        "name": "[parameters('storageAccountType')]"
    }
}
```

You must also specify the `AzureWebJobsStorage` property as an app setting in the site configuration. If the function app doesn't use Application Insights for monitoring, it should also specify `AzureWebJobsDashboard` as an app setting.

The Azure Functions runtime uses the `AzureWebJobsStorage` connection string to create internal queues.  When Application Insights is not enabled, the runtime uses the `AzureWebJobsDashboard` connection string to log to Azure Table storage and power the **Monitor** tab in the portal.

These properties are specified in the `appSettings` collection in the `siteConfig` object:

```json
"appSettings": [
    {
        "name": "AzureWebJobsStorage",
        "value": "[concat('DefaultEndpointsProtocol=https;AccountName=', variables('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountid'),'2019-06-01').keys[0].value)]"
    },
    {
        "name": "AzureWebJobsDashboard",
        "value": "[concat('DefaultEndpointsProtocol=https;AccountName=', variables('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountid'),'2019-06-01').keys[0].value)]"
    }
]
```

### Application Insights

Application Insights is recommended for monitoring your function apps. The Application Insights resource is defined with the type **Microsoft.Insights/components** and the kind **web**:

```json
        {
            "apiVersion": "2015-05-01",
            "name": "[variables('appInsightsName')]",
            "type": "Microsoft.Insights/components",
            "kind": "web",
            "location": "[resourceGroup().location]",
            "tags": {
                "[concat('hidden-link:', resourceGroup().id, '/providers/Microsoft.Web/sites/', variables('functionAppName'))]": "Resource"
            },
            "properties": {
                "Application_Type": "web",
                "ApplicationId": "[variables('appInsightsName')]"
            }
        },
```

In addition, the instrumentation key needs to be provided to the function app using the `APPINSIGHTS_INSTRUMENTATIONKEY` application setting. This property is specified in the `appSettings` collection in the `siteConfig` object:

```json
"appSettings": [
    {
        "name": "APPINSIGHTS_INSTRUMENTATIONKEY",
        "value": "[reference(resourceId('microsoft.insights/components/', variables('appInsightsName')), '2015-05-01').InstrumentationKey]"
    }
]
```

### Hosting plan

The definition of the hosting plan varies, and can be one of the following:
* [Consumption plan](#consumption) (default)
* [Premium plan](#premium)
* [App Service plan](#app-service-plan)

### Function app

The function app resource is defined by using a resource of type **Microsoft.Web/sites** and kind **functionapp**:

```json
{
    "apiVersion": "2015-08-01",
    "type": "Microsoft.Web/sites",
    "name": "[variables('functionAppName')]",
    "location": "[resourceGroup().location]",
    "kind": "functionapp",
    "dependsOn": [
        "[resourceId('Microsoft.Storage/storageAccounts', variables('storageAccountName'))]",
        "[resourceId('Microsoft.Insights/components', variables('appInsightsName'))]"
    ]
}
```

> [!IMPORTANT]
> If you are explicitly defining a hosting plan, an additional item would be needed in the dependsOn array: `"[resourceId('Microsoft.Web/serverfarms', variables('hostingPlanName'))]"`

A function app must include these application settings:

| Setting name                 | Description                                                                               | Example values                        |
|------------------------------|-------------------------------------------------------------------------------------------|---------------------------------------|
| AzureWebJobsStorage          | A connection string to a storage account that the Functions runtime uses for internal queueing | See [Storage account](#storage)       |
| FUNCTIONS_EXTENSION_VERSION  | The version of the Azure Functions runtime                                                | `~3`                                  |
| FUNCTIONS_WORKER_RUNTIME     | The language stack to be used for functions in this app                                   | `dotnet`, `node`, `java`, `python`, or `powershell` |
| WEBSITE_NODE_DEFAULT_VERSION | Only needed if using the `node` language stack, specifies the [version](./functions-reference-node.md#node-version) to use              | `~14`                             |

These properties are specified in the `appSettings` collection in the `siteConfig` property:

```json
"properties": {
    "siteConfig": {
        "appSettings": [
            {
                "name": "AzureWebJobsStorage",
                "value": "[concat('DefaultEndpointsProtocol=https;AccountName=', variables('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountid'),'2019-06-01').keys[0].value)]"
            },
            {
                "name": "FUNCTIONS_WORKER_RUNTIME",
                "value": "node"
            },
            {
                "name": "WEBSITE_NODE_DEFAULT_VERSION",
                "value": "~14"
            },
            {
                "name": "FUNCTIONS_EXTENSION_VERSION",
                "value": "~3"
            }
        ]
    }
}
```

<a name="consumption"></a>

## Deploy on Consumption plan

The Consumption plan automatically allocates compute power when your code is running, scales out as necessary to handle load, and then scales in when code is not running. You don't have to pay for idle VMs, and you don't have to reserve capacity in advance. To learn more, see [Azure Functions scale and hosting](consumption-plan.md).

For a sample Azure Resource Manager template, see [Function app on Consumption plan].

### Create a Consumption plan

A Consumption plan doesn't need to be defined. One will automatically be created or selected on a per-region basis when you create the function app resource itself.

The Consumption plan is a special type of "serverfarm" resource. For Windows, you can specify it by using the `Dynamic` value for the `computeMode` and `sku` properties:

```json
{
   "type":"Microsoft.Web/serverfarms",
   "apiVersion":"2016-09-01",
   "name":"[variables('hostingPlanName')]",
   "location":"[resourceGroup().location]",
   "properties":{
      "name":"[variables('hostingPlanName')]",
      "computeMode":"Dynamic"
   },
   "sku":{
      "name":"Y1",
      "tier":"Dynamic",
      "size":"Y1",
      "family":"Y",
      "capacity":0
   }
}
```

> [!NOTE]
> The Consumption plan cannot be explicitly defined for Linux. It will be created automatically.

If you do explicitly define your Consumption plan, you will need to set the `serverFarmId` property on the app so that it points to the resource ID of the plan. You should ensure that the function app has a `dependsOn` setting for the plan as well.

### Create a function app

The settings required by a function app running in Consumption plan differ between Windows and Linux.

#### Windows

On Windows, a Consumption plan requires an another setting in the site configuration: [`WEBSITE_CONTENTAZUREFILECONNECTIONSTRING`](functions-app-settings.md#website_contentazurefileconnectionstring). This property configures the storage account where the function app code and configuration are stored.

```json
{
    "apiVersion": "2016-03-01",
    "type": "Microsoft.Web/sites",
    "name": "[variables('functionAppName')]",
    "location": "[resourceGroup().location]",
    "kind": "functionapp",
    "dependsOn": [
        "[resourceId('Microsoft.Storage/storageAccounts', variables('storageAccountName'))]"
    ],
    "properties": {
        "siteConfig": {
            "appSettings": [
                {
                    "name": "AzureWebJobsStorage",
                    "value": "[concat('DefaultEndpointsProtocol=https;AccountName=', variables('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountid'),'2019-06-01').keys[0].value)]"
                },
                {
                    "name": "WEBSITE_CONTENTAZUREFILECONNECTIONSTRING",
                    "value": "[concat('DefaultEndpointsProtocol=https;AccountName=', variables('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountid'),'2019-06-01').keys[0].value)]"
                },
                {
                    "name": "FUNCTIONS_WORKER_RUNTIME",
                    "value": "node"
                },
                {
                    "name": "WEBSITE_NODE_DEFAULT_VERSION",
                    "value": "~14"
                },
                {
                    "name": "FUNCTIONS_EXTENSION_VERSION",
                    "value": "~3"
                }
            ]
        }
    }
}
```

> [!IMPORTANT]
> Don't set the [`WEBSITE_CONTENTSHARE`](functions-app-settings.md#website_contentshare) setting in a deployment slot. This setting is generated for you when the app is created in the deployment slot.

#### Linux

On Linux, the function app must have its `kind` set to `functionapp,linux`, and it must have the `reserved` property set to `true`.

```json
{
    "apiVersion": "2016-03-01",
    "type": "Microsoft.Web/sites",
    "name": "[variables('functionAppName')]",
    "location": "[resourceGroup().location]",
    "kind": "functionapp,linux",
    "dependsOn": [
        "[resourceId('Microsoft.Storage/storageAccounts', variables('storageAccountName'))]"
    ],
    "properties": {
        "siteConfig": {
            "appSettings": [
                {
                    "name": "AzureWebJobsStorage",
                    "value": "[concat('DefaultEndpointsProtocol=https;AccountName=', variables('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountName'),'2019-06-01').keys[0].value)]"
                },
                {
                    "name": "FUNCTIONS_WORKER_RUNTIME",
                    "value": "node"
                },
                {
                    "name": "WEBSITE_NODE_DEFAULT_VERSION",
                    "value": "~14"
                },
                {
                    "name": "FUNCTIONS_EXTENSION_VERSION",
                    "value": "~3"
                }
            ]
        },
        "reserved": true
    }
}
```

The [`WEBSITE_CONTENTAZUREFILECONNECTIONSTRING`](functions-app-settings.md#website_contentazurefileconnectionstring) and [`WEBSITE_CONTENTSHARE`](functions-app-settings.md#website_contentshare) settings aren't supported on Linux.

<a name="premium"></a>
## Deploy on Premium plan

The Premium plan offers the same scaling as the Consumption plan but includes dedicated resources and extra capabilities. To learn more, see [Azure Functions Premium Plan](./functions-premium-plan.md).

### Create a Premium plan

A Premium plan is a special type of "serverfarm" resource. You can specify it by using either `EP1`, `EP2`, or `EP3` for the `Name` property value in the `sku` [description object](/azure/templates/microsoft.web/2018-02-01/serverfarms#skudescription-object).

```json
{
    "type": "Microsoft.Web/serverfarms",
    "apiVersion": "2018-02-01",
    "name": "[parameters('hostingPlanName')]",
    "location": "[resourceGroup().location]",
    "properties": {
        "name": "[parameters('hostingPlanName')]",
        "workerSize": "[parameters('workerSize')]",
        "workerSizeId": "[parameters('workerSizeId')]",
        "numberOfWorkers": "[parameters('numberOfWorkers')]",
        "hostingEnvironment": "[parameters('hostingEnvironment')]",
        "maximumElasticWorkerCount": "20"
    },
    "sku": {
        "Tier": "ElasticPremium",
        "Name": "EP1"
    }
}
```

### Create a function app

A function app on a Premium plan must have the `serverFarmId` property set to the resource ID of the plan created earlier. In addition, a Premium plan requires an extra setting in the site configuration: [`WEBSITE_CONTENTAZUREFILECONNECTIONSTRING`](functions-app-settings.md#website_contentazurefileconnectionstring). This property configures the storage account where the function app code and configuration are stored.

```json
{
    "apiVersion": "2016-03-01",
    "type": "Microsoft.Web/sites",
    "name": "[variables('functionAppName')]",
    "location": "[resourceGroup().location]",
    "kind": "functionapp",
    "dependsOn": [
        "[resourceId('Microsoft.Web/serverfarms', variables('hostingPlanName'))]",
        "[resourceId('Microsoft.Storage/storageAccounts', variables('storageAccountName'))]"
    ],
    "properties": {
        "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', variables('hostingPlanName'))]",
        "siteConfig": {
            "appSettings": [
                {
                    "name": "AzureWebJobsStorage",
                    "value": "[concat('DefaultEndpointsProtocol=https;AccountName=', variables('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountid'),'2019-06-01').keys[0].value)]"
                },
                {
                    "name": "WEBSITE_CONTENTAZUREFILECONNECTIONSTRING",
                    "value": "[concat('DefaultEndpointsProtocol=https;AccountName=', variables('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountid'),'2019-06-01').keys[0].value)]"
                },
                {
                    "name": "FUNCTIONS_WORKER_RUNTIME",
                    "value": "node"
                },
                {
                    "name": "WEBSITE_NODE_DEFAULT_VERSION",
                    "value": "~14"
                },
                {
                    "name": "FUNCTIONS_EXTENSION_VERSION",
                    "value": "~3"
                }
            ]
        }
    }
}
```
> [!IMPORTANT]
> Don't set the [`WEBSITE_CONTENTSHARE`](functions-app-settings.md#website_contentshare) setting as it's generated for you when the site is first created.

<a name="app-service-plan"></a>

## Deploy on App Service plan

In the App Service plan, your function app runs on dedicated VMs on Basic, Standard, and Premium SKUs, similar to web apps. For details about how the App Service plan works, see the [Azure App Service plans in-depth overview](../app-service/overview-hosting-plans.md).

For a sample Azure Resource Manager template, see [Function app on Azure App Service plan].

### Create an App Service plan

An App Service plan is defined by a "serverfarm" resource.

```json
{
    "type": "Microsoft.Web/serverfarms",
    "apiVersion": "2018-02-01",
    "name": "[variables('hostingPlanName')]",
    "location": "[resourceGroup().location]",
    "sku": {
        "name": "S1",
        "tier": "Standard",
        "size": "S1",
        "family": "S",
        "capacity": 1
    }
}
```

To run your app on Linux, you must also set the `kind` to `Linux`:

```json
{
    "type": "Microsoft.Web/serverfarms",
    "apiVersion": "2018-02-01",
    "name": "[variables('hostingPlanName')]",
    "location": "[resourceGroup().location]",
    "kind": "Linux",
    "sku": {
        "name": "S1",
        "tier": "Standard",
        "size": "S1",
        "family": "S",
        "capacity": 1
    }
}
```

### Create a function app

A function app on an App Service plan must have the `serverFarmId` property set to the resource ID of the plan created earlier.

```json
{
    "apiVersion": "2016-03-01",
    "type": "Microsoft.Web/sites",
    "name": "[variables('functionAppName')]",
    "location": "[resourceGroup().location]",
    "kind": "functionapp",
    "dependsOn": [
        "[resourceId('Microsoft.Web/serverfarms', variables('hostingPlanName'))]",
        "[resourceId('Microsoft.Storage/storageAccounts', variables('storageAccountName'))]"
    ],
    "properties": {
        "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', variables('hostingPlanName'))]",
        "siteConfig": {
            "appSettings": [
                {
                    "name": "AzureWebJobsStorage",
                    "value": "[concat('DefaultEndpointsProtocol=https;AccountName=', variables('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountid'),'2019-06-01').keys[0].value)]"
                },
                {
                    "name": "FUNCTIONS_WORKER_RUNTIME",
                    "value": "node"
                },
                {
                    "name": "WEBSITE_NODE_DEFAULT_VERSION",
                    "value": "~14"
                },
                {
                    "name": "FUNCTIONS_EXTENSION_VERSION",
                    "value": "~3"
                }
            ]
        }
    }
}
```

Linux apps should also include a `linuxFxVersion` property under `siteConfig`. If you are just deploying code, the value for this is determined by your desired runtime stack in the format of ```runtime|runtimeVersion```:

| Stack            | Example value                                         |
|------------------|-------------------------------------------------------|
| Python           | `python\|3.7`      |
| JavaScript       | `node\|14`          |
| .NET             | `dotnet\|3.1` |

```json
{
    "apiVersion": "2016-03-01",
    "type": "Microsoft.Web/sites",
    "name": "[variables('functionAppName')]",
    "location": "[resourceGroup().location]",
    "kind": "functionapp",
    "dependsOn": [
        "[resourceId('Microsoft.Web/serverfarms', variables('hostingPlanName'))]",
        "[resourceId('Microsoft.Storage/storageAccounts', variables('storageAccountName'))]"
    ],
    "properties": {
        "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', variables('hostingPlanName'))]",
        "siteConfig": {
            "appSettings": [
                {
                    "name": "AzureWebJobsStorage",
                    "value": "[concat('DefaultEndpointsProtocol=https;AccountName=', variables('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountid'),'2019-06-01').keys[0].value)]"
                },
                {
                    "name": "FUNCTIONS_WORKER_RUNTIME",
                    "value": "node"
                },
                {
                    "name": "WEBSITE_NODE_DEFAULT_VERSION",
                    "value": "~14"
                },
                {
                    "name": "FUNCTIONS_EXTENSION_VERSION",
                    "value": "~3"
                }
            ],
            "linuxFxVersion": "node|14"
        }
    }
}
```

If you are [deploying a custom container image](./functions-create-function-linux-custom-image.md), you must specify it with `linuxFxVersion` and include configuration that allows your image to be pulled, as in [Web App for Containers](../app-service/index.yml). Also, set `WEBSITES_ENABLE_APP_SERVICE_STORAGE` to `false`, since your app content is provided in the container itself:

```json
{
    "apiVersion": "2016-03-01",
    "type": "Microsoft.Web/sites",
    "name": "[variables('functionAppName')]",
    "location": "[resourceGroup().location]",
    "kind": "functionapp",
    "dependsOn": [
        "[resourceId('Microsoft.Web/serverfarms', variables('hostingPlanName'))]",
        "[resourceId('Microsoft.Storage/storageAccounts', variables('storageAccountName'))]"
    ],
    "properties": {
        "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', variables('hostingPlanName'))]",
        "siteConfig": {
            "appSettings": [
                {
                    "name": "AzureWebJobsStorage",
                    "value": "[concat('DefaultEndpointsProtocol=https;AccountName=', variables('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountid'),'2019-06-01').keys[0].value)]"
                },
                {
                    "name": "FUNCTIONS_WORKER_RUNTIME",
                    "value": "node"
                },
                {
                    "name": "WEBSITE_NODE_DEFAULT_VERSION",
                    "value": "~14"
                },
                {
                    "name": "FUNCTIONS_EXTENSION_VERSION",
                    "value": "~3"
                },
                {
                    "name": "DOCKER_REGISTRY_SERVER_URL",
                    "value": "[parameters('dockerRegistryUrl')]"
                },
                {
                    "name": "DOCKER_REGISTRY_SERVER_USERNAME",
                    "value": "[parameters('dockerRegistryUsername')]"
                },
                {
                    "name": "DOCKER_REGISTRY_SERVER_PASSWORD",
                    "value": "[parameters('dockerRegistryPassword')]"
                },
                {
                    "name": "WEBSITES_ENABLE_APP_SERVICE_STORAGE",
                    "value": "false"
                }
            ],
            "linuxFxVersion": "DOCKER|myacr.azurecr.io/myimage:mytag"
        }
    }
}
```

## Deploy to Azure Arc

Azure Functions can be deployed to [Azure Arc-enabled Kubernetes](../app-service/overview-arc-integration.md). This process largely follows [deploying to an App Service plan](#deploy-on-app-service-plan), with a few differences to note.

To create the app and plan resources, you must have already [created an App Service Kubernetes environment](../app-service/manage-create-arc-environment.md) for an Azure Arc-enabled Kubernetes cluster. These examples assume you have the resource ID of the custom location and App Service Kubernetes environment that you are deploying to. For most templates, you can supply these as parameters.

```json
{
    "parameters": {
        "kubeEnvironmentId" : {
            "type": "string"
        },
        "customLocationId" : {
            "type": "string"
        }
    }
}
```

Both sites and plans must reference the custom location through an `extendedLocation` field. This block sits outside of `properties`, peer to `kind` and `location`:

```json
{
    "extendedLocation": {
        "type": "customlocation",
        "name": "[parameters('customLocationId')]"
    },
}
```

The plan resource should use the Kubernetes (K1) SKU, and its `kind` field should be "linux,kubernetes". Within `properties`, `reserved` should be "true", and `kubeEnvironmentProfile.id` should be set to the App Service Kubernetes environment resource ID. An example plan might look like the following:

```json
{
    "type": "Microsoft.Web/serverfarms",
    "name": "[variables('hostingPlanName')]",
    "location": "[parameters('location')]",
    "apiVersion": "2020-12-01",
    "kind": "linux,kubernetes",
    "sku": {
        "name": "K1",
        "tier": "Kubernetes"
    },
    "extendedLocation": {
        "type": "customlocation",
        "name": "[parameters('customLocationId')]"
    },
    "properties": {
        "name": "[variables('hostingPlanName')]",
        "location": "[parameters('location')]",
        "workerSizeId": "0",
        "numberOfWorkers": "1",
        "kubeEnvironmentProfile": {
            "id": "[parameters('kubeEnvironmentId')]"
        },
        "reserved": true
    }
}
```

The function app resource should have its `kind` field set to "functionapp,linux,kubernetes" or "functionapp,linux,kubernetes,container" depending on if you intend to deploy via code or container. An example function app might look like the following:

```json
 {
    "apiVersion": "2018-11-01",
    "type": "Microsoft.Web/sites",
    "name": "[variables('appName')]",
    "kind": "kubernetes,functionapp,linux,container",
    "location": "[parameters('location')]",
    "extendedLocation": {
        "type": "customlocation",
        "name": "[parameters('customLocationId')]"
    },
    "dependsOn": [
        "[resourceId('Microsoft.Insights/components', variables('appInsightsName'))]",
        "[resourceId('Microsoft.Storage/storageAccounts', variables('storageAccountName'))]",
        "[variables('hostingPlanId')]"
    ],
    "properties": {
        "serverFarmId": "[variables('hostingPlanId')]",
        "siteConfig": {
            "linuxFxVersion": "DOCKER|mcr.microsoft.com/azure-functions/dotnet:3.0-appservice-quickstart",
            "appSettings": [
                {
                    "name": "FUNCTIONS_EXTENSION_VERSION",
                    "value": "~3"
                },
                {
                    "name": "AzureWebJobsStorage",
                    "value": "[concat('DefaultEndpointsProtocol=https;AccountName=', variables('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountid'),'2015-05-01-preview').key1)]"

                },
                {
                    "name": "APPINSIGHTS_INSTRUMENTATIONKEY",
                    "value": "[reference(resourceId('microsoft.insights/components/', variables('appInsightsName')), '2015-05-01').InstrumentationKey]"
                }
            ],
            "alwaysOn": true
        }
    }
}
```

## Customizing a deployment

A function app has many child resources that you can use in your deployment, including app settings and source control options. You also might choose to remove the **sourcecontrols** child resource, and use a different [deployment option](functions-continuous-deployment.md) instead.

> [!IMPORTANT]
> To successfully deploy your application by using Azure Resource Manager, it's important to understand how resources are deployed in Azure. In the following example, top-level configurations are applied by using **siteConfig**. It's important to set these configurations at a top level, because they convey information to the Functions runtime and deployment engine. Top-level information is required before the child **sourcecontrols/web** resource is applied. Although it's possible to configure these settings in the child-level **config/appSettings** resource, in some cases your function app must be deployed *before* **config/appSettings** is applied. For example, when you are using functions with [Logic Apps](../logic-apps/index.yml), your functions are a dependency of another resource.

```json
{
  "apiVersion": "2015-08-01",
  "name": "[parameters('appName')]",
  "type": "Microsoft.Web/sites",
  "kind": "functionapp",
  "location": "[parameters('location')]",
  "dependsOn": [
    "[resourceId('Microsoft.Storage/storageAccounts', variables('storageAccountName'))]",
    "[resourceId('Microsoft.Web/serverfarms', parameters('appName'))]"
  ],
  "properties": {
     "serverFarmId": "[variables('appServicePlanName')]",
     "siteConfig": {
        "alwaysOn": true,
        "appSettings": [
            {
                "name": "FUNCTIONS_EXTENSION_VERSION",
                "value": "~3"
            },
            {
                "name": "Project",
                "value": "src"
            }
        ]
     }
  },
  "resources": [
     {
        "apiVersion": "2015-08-01",
        "name": "appsettings",
        "type": "config",
        "dependsOn": [
          "[resourceId('Microsoft.Web/Sites', parameters('appName'))]",
          "[resourceId('Microsoft.Web/Sites/sourcecontrols', parameters('appName'), 'web')]",
          "[resourceId('Microsoft.Storage/storageAccounts', variables('storageAccountName'))]"
        ],
        "properties": {
          "AzureWebJobsStorage": "[concat('DefaultEndpointsProtocol=https;AccountName=', variables('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountid'),'2019-06-01').keys[0].value)]",
          "AzureWebJobsDashboard": "[concat('DefaultEndpointsProtocol=https;AccountName=', variables('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountid'),'2019-06-01').keys[0].value)]",
          "FUNCTIONS_EXTENSION_VERSION": "~3",
          "FUNCTIONS_WORKER_RUNTIME": "dotnet",
          "Project": "src"
        }
     },
     {
          "apiVersion": "2015-08-01",
          "name": "web",
          "type": "sourcecontrols",
          "dependsOn": [
            "[resourceId('Microsoft.Web/sites/', parameters('appName'))]"
          ],
          "properties": {
            "RepoUrl": "[parameters('sourceCodeRepositoryURL')]",
            "branch": "[parameters('sourceCodeBranch')]",
            "IsManualIntegration": "[parameters('sourceCodeManualIntegration')]"
          }
     }
  ]
}
```
> [!TIP]
> This template uses the [Project](https://github.com/projectkudu/kudu/wiki/Customizing-deployments#using-app-settings-instead-of-a-deployment-file) app settings value, which sets the base directory in which the Functions deployment engine (Kudu) looks for deployable code. In our repository, our functions are in a subfolder of the **src** folder. So, in the preceding example, we set the app settings value to `src`. If your functions are in the root of your repository, or if you are not deploying from source control, you can remove this app settings value.

## Deploy your template

You can use any of the following ways to deploy your template:

* [PowerShell](../azure-resource-manager/templates/deploy-powershell.md)
* [Azure CLI](../azure-resource-manager/templates/deploy-cli.md)
* [Azure portal](../azure-resource-manager/templates/deploy-portal.md)
* [REST API](../azure-resource-manager/templates/deploy-rest.md)

### Deploy to Azure button

Replace ```<url-encoded-path-to-azuredeploy-json>``` with a [URL-encoded](https://www.bing.com/search?q=url+encode) version of the raw path of your `azuredeploy.json` file in GitHub.

Here is an example that uses markdown:

```markdown
[![Deploy to Azure](https://azuredeploy.net/deploybutton.png)](https://portal.azure.com/#create/Microsoft.Template/uri/<url-encoded-path-to-azuredeploy-json>)
```

Here is an example that uses HTML:

```html
<a href="https://portal.azure.com/#create/Microsoft.Template/uri/<url-encoded-path-to-azuredeploy-json>" target="_blank"><img src="https://azuredeploy.net/deploybutton.png"></a>
```

### Deploy using PowerShell

The following PowerShell commands create a resource group and deploy a template that creates a function app with its required resources. To run locally, you must have [Azure PowerShell](/powershell/azure/install-az-ps) installed. Run [`Connect-AzAccount`](/powershell/module/az.accounts/connect-azaccount) to sign in.

```powershell
# Register Resource Providers if they're not already registered
Register-AzResourceProvider -ProviderNamespace "microsoft.web"
Register-AzResourceProvider -ProviderNamespace "microsoft.storage"

# Create a resource group for the function app
New-AzResourceGroup -Name "MyResourceGroup" -Location 'West Europe'

# Create the parameters for the file, which for this template is the function app name.
$TemplateParams = @{"appName" = "<function-app-name>"}

# Deploy the template
New-AzResourceGroupDeployment -ResourceGroupName "MyResourceGroup" -TemplateFile template.json -TemplateParameterObject $TemplateParams -Verbose
```

To test out this deployment, you can use a [template like this one](https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/quickstarts/microsoft.web/function-app-create-dynamic/azuredeploy.json) that creates a function app on Windows in a Consumption plan. Replace `<function-app-name>` with a unique name for your function app.

## Next steps

Learn more about how to develop and configure Azure Functions.

* [Azure Functions developer reference](functions-reference.md)
* [How to configure Azure function app settings](functions-how-to-use-azure-function-app-settings.md)
* [Create your first Azure function](./functions-get-started.md)

<!-- LINKS -->

[Function app on Consumption plan]: https://azure.microsoft.com/resources/templates/function-app-create-dynamic/
[Function app on Azure App Service plan]: https://azure.microsoft.com/resources/templates/function-app-create-dedicated/