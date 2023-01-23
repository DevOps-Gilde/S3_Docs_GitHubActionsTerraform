# Monitoring

## General

This is an optional extra task you can work on. Focus of the task is monitoring your demo web app. App Service is deeply integrated with App Insights. App Insights is a feature within Azure Monitor. To monitor your Web App you have to create a new App Insights resource that is linked to use the **[Kusto query language](https://docs.microsoft.com/de-de/azure/data-explorer/kusto/concepts/)** to retrieve the **telemetry** data. The picture below illustrates this:
<br><img src="./images/mon_overview.png" width="400"/>

To watch your **telemetry** in App Insight we will use the portal. We created special users for portal access. Contact one of us to get the credentials for it. The next two big steps are:
1. Extend your infra pipeline to create an App Insights resource and link it to your Web App
2. Monitor your Web App via App Insights resource
3. Monitor your Web App via a dashboard

The next chapters will detail these steps.

## Extend your infra pipeline

To create the additional components we will add additional code to the already existing terraform module `main.tf`.

Application insights started as a separate stand alone resource. In the meantime Microsoft recommends a setup together with a log analytics workspace. In this scenario application insights references a log analytics workspace. Therefore, as a first step we must create the log analytics workspace before we can continue. To achieve that add the following declaration to the task outlined previously. Replace `<your prefix>` with your unique prefix since all participants are working in the same resource group:
```
resource "azurerm_log_analytics_workspace" "log" {
  name                = "<your prefix>-lg-analytics"
  location            = data.azurerm_resource_group.wsdevops.location
  resource_group_name = data.azurerm_resource_group.wsdevops.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}
```

Now we can finally setup the application insight resource. The link between both is established by setting the property `workspace_id`. Replace `<your prefix>` with a unique prefix since all participants are working in the same resource group:
```
resource "azurerm_application_insights" "appi" {
  name                = "<your prefix>-api"
  location            = data.azurerm_resource_group.wsdevops.location
  resource_group_name = data.azurerm_resource_group.wsdevops.name
  workspace_id        = azurerm_log_analytics_workspace.log.id
  application_type    = "web"
}
```

Now we will link your application insights resource with your web app. In our last session we used Azure CLI for that. We will stick to it, although we could also achieve that in terraform. This means we will have to embed scripting code in terraform modules. Why is that pattern important? It takes some time until new features are implemented in terraform. Therefore, falling back to scripting is something that can hit you easily. The first challenge is to provide a special null resource where our scripting code lives in. Below you find a first skeleton:
```
// Skeleton for linking web app and app insights
resource "null_resource" "link_monitoring" {
  provisioner "local-exec" {
    command = <<EOT
      # Login to Azure CLI (Linux operating system assumed)
      az login --service-principal -u $con_client_id -p $con_client_secret --tenant $con_tenant_id
      # TODO your scripting code
    EOT
    environment = {
      // Parameters needed to login
      con_client_id     = TODO
      con_client_secret = TODO
      con_tenant_id     = TODO
      // Parameters needed for linking
      inst_key          = TODO
      conn_str          = TODO      
      rg_name           = TODO
      web_app_name      = TODO
    }
  }
}
```
First a few comments regarding the blocks:
1. command

   Holds your script code marked by the keyword `EOT` inline. Note that the script code is a black box for terraform. Any dependencies beyond the input parameters have to be stated explicitly on the resource level.
2. environment

   This section is used to pass terraform values to your scripting code as environment variables. The lefthand side states the name and righthand side states the terraform expression used as value. The environment variables can be used by the scripting code according to the conventions by the used scripting language. The conventions depend on the VM operating system you are using.

As you have seen there are a few `TODO`s left to fill. Let's start with the parameters. The first group results from the required `az login` command. It requires (1) the id of the current service principal, (2) the tenant id and (3) the secret of the service principal. 
All three can be obtained from the variables we introduced earlier. To reference a variable you have to state `var.<name-of-variable>`. The instrumentation key can be obtained from the created application insight resource with `azurerm_application_insights.appi.instrumentation_key`. The value for the connection string is `azurerm_application_insights.appi.connection_string`. The resource group name follows the same pattern as applied earlier.
The last missing piece is our scripting code. Place the following code below the az login command:
```
az webapp config appsettings set --name $web_app_name --resource-group $rg_name --settings APPINSIGHTS_INSTRUMENTATIONKEY=$inst_key APPINSIGHTS_PROFILERFEATURE_VERSION=1.0.0 APPINSIGHTS_SNAPSHOTFEATURE_VERSION=1.0.0 APPLICATIONINSIGHTS_CONNECTION_STRING=$conn_str ApplicationInsightsAgent_EXTENSION_VERSION=~3 DiagnosticServices_EXTENSION_VERSION=~3 InstrumentationEngine_EXTENSION_VERSION=disabled SnapshotDebugger_EXTENSION_VERSION=disabled XDT_MicrosoftApplicationInsights_BaseExtensions=recommended XDT_MicrosoftApplicationInsights_PreemptSdk=disabled
```
This command adds application settings to our web application. They constitute the link so that telemetry is forwarded.

## Run your extended Infrastructure Pipeline

Now it is time to run our infrastructure pipeline. After the pipeline run you should have additional app insights and log analytic resources. The pipeline run might take up to 6 minutes.

## Monitor via App Insights resource

First contact one of us to grab a user along with the credentials.

To login to the Azure Portal enter "portal.azure.com" in the browser. Use the credentials given to you to login.

Navigate in the portal to the resource group "ws-devops".
<br><img src="./images/mon_appi_rg.png" width="400"/>

Open your App Insights instance and by clicking on the entry representing your instance. Click in the main menu on the left hand side on "Logs". Click away the welcome and query screen.
<br><img src="./images/mon_appi_logs.png" width="400"/>

You will see detail screen where you can enter your Kusto query. The screenshot below explains a bit the major controls.
<br><img src="./images/mon_appi_kusto.png" width="400"/>

The simplest Kusto query is just to state the name of the log table you are interested in as shown below. If you execute this query by pressing the previously outlined button you get the result. Note that there is a small delay until the data is visible in App Insights (app. 1 minute).
```
requests
```

Let's run now a bit more sophisticated query that counts all failed requests (In our case all requests where the HTTP status code does not equal 200). We will extend our query by two keywords:
1. where - as in SQL this allows us to filter the records by a certain criteria
2. summarize - Allows you to run an aggregating operation such as count

The snippet below shows the full query. As you can see both keywords are chained by the pipe operator:
```
requests
| where resultCode != 200
| summarize count()
```
The result of the query should output the simplified number of failed requests now. With these two queries we just touched the basics of monitoring in App Insights. Kusto provides much more capabilities and you can also use it to trigger automatic actions based on the query result.

## Monitor via dashboard

In reality you have to monitor a lot of places. A dashboard gives you a chance to collect all relevant information in a single place. We will now integrate our query now in a first example dashboard.
From terraform perspective a dashboard is just another resource. The complete layout along with its content is stored as JSON in one of the resource properties. The JSON is internal. Hardcoding the content like queries and other settings would make the code hard to maintain. Therefore, we have to implement two steps:
1. Template for the dashboard
2. Creating dashboard resource and assign template with values in your existing `main.tf` file

**Regarding template)**

We created already a file `dashboard.tpl` that contains the internal JSON. We added already placeholders to avoid hardcoded parameters. We now have to make sure that terraform sets the correct values. We will need (1) the name of the created application insights resource, (2) the name of the resource group, (3) the current subscription id and (4) our query. All values are passed in the same fashion as already before.
The code below now shows how you can make the JSON available for later use:
```
data "template_file" "dash-template" {
  template = file("${path.module}/dashboard.tpl")
  vars = {
    api_name = azurerm_application_insights.appi.name
    rg_name  = data.azurerm_resource_group.wsdevops.name
    sub_id   = var.subscription_id
    query    = "requests | where resultCode != 200 | summarize count()"
  }
}
```
Note the following interesting points:
1. template property

   The expression constructs the file path for the template file. `${path.module}`resolves to the current path within the current tf-module. We assume that `dashboard.tpl` is located in the same directory as `main.tf`. `file`concatenates both parts and yields the final path.
2. Vars section

   The variables are defined in the var section such as `sub_id`. The standard expression syntax `${sub_id}` must be used inside the file `dashboard.tpl` to trigger the replacement with the value.

**Creating dashboard resource)**

Now we can reference the template file in the dashboard resource as follows. As above use a unique prefix since you are all working in the same resource group:
```
resource "azurerm_dashboard" "my-board" {
  name                = "<your prefix>-dashboard"
  resource_group_name = data.azurerm_resource_group.wsdevops.name
  location            = data.azurerm_resource_group.wsdevops.location
  tags = {
    source = "terraform"
  }
  dashboard_properties = data.template_file.dash-template.rendered
}
```
`data.template_file.dash-template` follows the usual pattern to reference existing resources (data = since existing; type = template_file; dash-template = user specified name). `Rendered` is the property name defined by terraform to retrieve the JSON with the replaced values.

There are multiple ways to open the dashboard you created. One way is to open the resource group `ws-devops` in the portal. There you should have an entry for your dashboard `<your prefix>-dashboard`. Click on the entry. In the overview page you have a link as shown below that taks you directly to your dashboard.
