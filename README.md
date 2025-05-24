# Azure Functions in Python (Isolated Process) — End-to-End Guide

This project demonstrates how to build Python-based Azure Functions using the **Isolated Process model** in Visual Studio Code, including:

1. Creating your first Azure Function.
2. Adding output bindings for **Azure Storage Queue** and **Azure SQL Database**.
3. Testing and deploying your app using Visual Studio Code.

------

## 🧰 Prerequisites

Ensure the following tools are installed:

- [Python 3.7+](https://www.python.org/downloads/)
- [Visual Studio Code](https://code.visualstudio.com/)
  - Azure Functions Extension
  - Python Extension
  - Azure Storage Extension
  - Azurite Extension (optional, for local development)
- [.NET SDK](https://dotnet.microsoft.com/download)
- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
- [Azure Functions Core Tools](https://learn.microsoft.com/en-us/azure/azure-functions/functions-run-local)
- An active [Azure subscription](https://azure.microsoft.com/free)

------

## ① Create Your First Azure Function

### Step-by-Step

1. Open VS Code → `F1` → `Azure Functions: Create New Project...`

2. Select folder and choose:

   - **Language**: Python (Isolated Process)
   - **Trigger**: HTTP trigger
   - **Function Name**: `HttpExample`
   - **Authorization Level**: Anonymous

3. Select Python interpreter and open the project.

4. In the local.settings.json file, update the `AzureWebJobsStorage` setting as in the following example:

   ```json
   "AzureWebJobsStorage": "UseDevelopmentStorage=true",
   ```

### Start Azurite Emulator (Optional)

In Visual Studio Code, press F1 to open the command palette. In the command palette, search for and select:

```bash
Azurite: Start
```

Check the bottom bar and verify that Azurite emulation services are running. If so, you can now run your function locally.

### Run Locally

```
F5
```

Trigger the function via:

- Azure extension → Local Project > Functions > HttpExample → Execute Function Now
- Use this JSON input:

```json
{
  "name": "Azure"
}
```

------

## ② Add Output Binding to Azure Storage Queue

### Download the function app settings

1. Press F1 to open the command palette, then search for and run the command `Azure Functions: Download Remote Settings...`.
2. Choose the function app you created in the previous article. Select **Yes to all** to overwrite the existing local settings.Update Python Code
3. Copy the value `AzureWebJobsStorage`, which is the key for the storage account connection string value. You use this connection to verify that the output binding works as expected.

### Register binding extensions

Your project has been configured to use [extension bundles](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-register#extension-bundles), which automatically installs a predefined set of extension packages.

Extension bundles is already enabled in the *host.json* file at the root of the project, which should look like the following example:

JSON

```json
{
  "version": "2.0",
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[3.*, 4.0.0)"
  }
}
```

Now, you can add the storage output binding to your project.

### Add code that uses the output binding

In `function_app.py`:

```
import azure.functions as func
import logging

app = func.FunctionApp(http_auth_level=func.AuthLevel.ANONYMOUS)

@app.route(route="HttpExample")
@app.queue_output(arg_name="msg", queue_name="outqueue", connection="AzureWebJobsStorage")
def HttpExample(req: func.HttpRequest, msg: func.Out [func.QueueMessage]) -> func.HttpResponse:
    logging.info('Python HTTP trigger function processed a request.')

    name = req.params.get('name')
    if not name:
        try:
            req_body = req.get_json()
        except ValueError:
            pass
        else:
            name = req_body.get('name')

    if name:
        msg.set(name)
        return func.HttpResponse(f"Hello, {name}. This HTTP triggered function executed successfully.")
    else:
        return func.HttpResponse(
             "This HTTP triggered function executed successfully. Pass a name in the query string or in the request body for a personalized response.",
             status_code=200
        )
```

### Run the function locally

1.  Press F5 to start the function app project and Core Tools.
2. With the Core Tools running, go to the **Azure: Functions** area. Under **Functions**, expand **Local Project** > **Functions**. Right-click (Ctrl-click on Mac) the `HttpExample` function and select **`Execute Function Now...`**.
3. In the **Enter request body**, you see the request message body value of `{ "name": "Azure" }`. Press Enter to send this request message to your function.
4. After a response is returned, press Ctrl + C to stop Core Tools.

### Connect Storage Explorer to your account

Skip this section if you've already installed Azure Storage Explorer and connected it to your Azure account.

1. Run the [Azure Storage Explorer](https://storageexplorer.com/) tool, select the connect icon on the left, and select **Add an account**.
2. In the **Connect** dialog, choose **Add an Azure account**, choose your **Azure environment**, and then select **`Sign in...`**.
3. After you successfully sign in to your account, you see all of the Azure subscriptions associated with your account. Choose your subscription and select **Open Explorer**.

### Examine the output queue

1. In Visual Studio Code, press F1 to open the command palette, then search for and run the command `Azure Storage: Open in Storage Explorer` and choose your storage account name. Your storage account opens in the Azure Storage Explorer.

2. Expand the **Queues** node, and then select the queue named **outqueue**.

   The queue contains the message that the queue output binding created when you ran the HTTP-triggered function. If you invoked the function with the default `name` value of *Azure*, the queue message is *Name passed to the function: Azure*.

3. Run the function again, send another request, and you see a new message in the queue.

### Redeploy and verify the updated app

1. In Visual Studio Code, press F1 to open the command palette. In the command palette, search for and select `Azure Functions: Deploy to function app...`.
2. Choose the function app that you created in the first article. Because you're redeploying your project to the same app, select **Deploy** to dismiss the warning about overwriting files.
3. After the deployment completes, you can again use the **Execute Function Now...** feature to trigger the function in Azure.
4. Again [view the message in the storage queue](https://learn.microsoft.com/en-us/azure/azure-functions/functions-add-output-binding-storage-queue-vs-code?pivots=programming-language-python&tabs=isolated-process#examine-the-output-queue) to verify that the output binding generates a new message in the queue.

------

## ③ Add Output Binding to Azure SQL Database

### Setup Azure SQL

1. [Create Azure SQL Database](https://learn.microsoft.com/en-us/azure/azure-sql/database/single-database-create-quickstart)
2. Create table:

```
sqlCopyEditCREATE TABLE dbo.ToDo (
    [Id] UNIQUEIDENTIFIER PRIMARY KEY,
    [order] INT NULL,
    [title] NVARCHAR(200) NOT NULL,
    [url] NVARCHAR(200) NOT NULL,
    [completed] BIT NOT NULL
);
```

1. Copy ADO.NET connection string from portal → SQL Authentication.

### Add Connection String

In Azure:

- App Settings → Add: `SqlConnectionString`

Download settings locally:

```
sh


CopyEdit
F1 → Azure Functions: Download Remote Settings...
```

### Install SQL Binding Extension

```
sh


CopyEdit
dotnet add package Microsoft.Azure.Functions.Worker.Extensions.Sql
```

In `host.json`:

```
jsonCopyEdit{
  "version": "2.0",
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.*, 5.0.0)"
  }
}
```

### Add Python SQL Binding

In `function_app.py`:

```
pythonCopyEdit@app.generic_output_binding(
    arg_name="outputTable",
    type="sql",
    command_text="dbo.ToDo",
    connection_string_setting="SqlConnectionString"
)
```

Update function to:

```
pythonCopyEditimport uuid

@app.function_name(name="HttpExample")
@app.route(route="todo")
@app.generic_output_binding(
    arg_name="outputTable",
    type="sql",
    command_text="dbo.ToDo",
    connection_string_setting="SqlConnectionString"
)
def main(req: func.HttpRequest) -> func.SqlOutput:
    req_body = req.get_json()
    todo = {
        "Id": str(uuid.uuid4()),
        "order": req_body.get("order"),
        "title": req_body.get("title"),
        "url": req_body.get("url"),
        "completed": req_body.get("completed")
    }
    return func.SqlOutput(table=todo)
```

### Test SQL Output

Trigger with:

```
jsonCopyEdit{
  "order": 1,
  "title": "Hello SQL",
  "url": "https://example.com",
  "completed": false
}
```

Check the `ToDo` table in your database.

------

## 🚀 Deployment

1. `F1 → Azure Functions: Deploy to Function App...`
2. Select your Function App.
3. Confirm deployment.
4. Run functions in Azure via "Execute Function Now".

------

## 🧹 Clean Up

To delete all resources:

1. `F1 → Azure: Open in Portal`
2. Go to the resource group
3. Click **Delete resource group**