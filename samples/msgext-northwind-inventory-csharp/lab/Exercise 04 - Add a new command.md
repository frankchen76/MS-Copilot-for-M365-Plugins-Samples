# Building plugins for Microsoft 365 Copilot

TABLE OF CONTENTS

* [Welcome](./Exercise%2000%20-%20Welcome.md) 
* [Exercise 1](./Exercise%2001%20-%20Set%20up.md) - Set up your development Environment 
* [Exercise 2](./Exercise%2002%20-%20Run%20sample%20app.md) - Run the sample as a Message Extension
* [Exercise 3](./Exercise%2003%20-%20Run%20in%20Copilot.md) - Run the sample as a Copilot plugin
* Exercise 4 - Add a new command **(THIS PAGE)**
* [Exercise 5](./Exercise%2005%20-%20Code%20tour.md) - Code tour

## Exercise 4 - Add a new command 

In this exercise, you will enhance the Teams Message Extension (ME) / Copilot plugin by adding a new command. While the current message extension effectively provides information about products within the Northwind inventory database, it does not provide information related to Northwind’s customers. Your task is to introduce a new command associated with an API call that retrieves products ordered by a customer name specified by the user. This exercise assumes you have completed at least exercises 1 and 2. It's fine to skip Exercise 3 in case you don't have a Microsoft 365 Copilot license.

To do this you'll complete five tasks.
1. **Extend the Message Extension / plugin User Interface** by modifying the Teams app manifest. This includes introducing a new command: **"companySearch"**. Note the UI for the Message Extension is an adaptive card where for Copilot it is text input and output in Copilot chat.
2. **Create a handler for the 'companySearch' command**. This will parse the query string passed in from the message routing code, validate the input and call the product search by company API. This step will also populate an adaptive card with the returned product list which will be displayed in the ME or Copilot chat UI.
3. Update the command **routing** code to route the new command to the handler created in the previous step. You'll do this by extending the method called by the Bot Framework when users query the Northwind database (**OnTeamsMessagingExtensionQueryAsync**). 
4. **Implement Product Search by Company** that returns a list of products ordered by that company.
5. **Run the app** and search of products that were purchased by a specified company.

# Step 1 - Extend the Message Extension / plugin User Interface 

1. Open **manifest.json** and add the following json immediately after the `discountSearch` command. Here you're adding to the `commands` array which defines the list of commands supported by the ME / plugin.

```json
{
    "id": "companySearch",
    "context": [
        "compose",
        "commandBox"
    ],
    "description": "Given a company name, search for products ordered by that company",
    "title": "Customer",
    "type": "query",
    "parameters": [
        {
            "name": "companyName",
            "title": "Company name",
            "description": "The company name to find products ordered by that company",
            "inputType": "text"
        }
    ]
}
```
```
Note: The "id" is the connection between the UI and the code. This value is defined as COMMAND_ID in the Bot\SearchBot.cs files. See how each of these files has a unique COMMAND_ID that corresponds to the value of "id".
```

# Step 2 - Create a handler for the 'companySearch' command
We will use a lot of the code created for the other handlers. 

1. In VS copy '**ProductSearchCommand.cs**' and paste into the same folder to create a copy. Rename this file **CustomerSearchCommand.cs**.

2. In **CustomerSearchCommand.cs**, after the line `public static class ProductSearchCommand`, replace the value of **CommandId** with "companySearch" as shown below:
```csharp
public const string CommandId = "companySearch";
```

3. Replace the content of **HandleTeamsMessagingExtensionQueryAsync** with:
```csharp
{
 string companyName;

// Validate the incoming query, making sure it's the 'companySearch' command
// The value of the 'companyName' parameter is the company name to search for
if (query.Parameters.Count == 1 && query.Parameters[0]?.Name == "companyName")
{
    var values = query.Parameters[0]?.Value.ToString().Split(',');
    companyName = values.ElementAtOrDefault(0);
}
else
{
    companyName = Utils.CleanupParam(query.Parameters?.FirstOrDefault(p => p.Name == "companyName")?.Value as string);
}

Console.WriteLine($"🍽️ Query #{++queryCount}:\ncompanyName={companyName}");

ProductService productService = new ProductService(configuration);
var products = await productService.SearchProductsByCustomer(companyName);

Console.WriteLine($"Found {products.Count} products in the Northwind database");
var attachments = new List<MessagingExtensionAttachment>();

foreach (var product in products)
{
    var preview = new HeroCard
    {
        Title = product.ProductName,
        Subtitle = $"Customer: {companyName}",
        Images = new List<CardImage> { new CardImage(product.ImageUrl) }
    }.ToAttachment();

    var resultCard = CardHandler.GetEditCard(product);

    var attachment = new MessagingExtensionAttachment
    {
        ContentType = resultCard.ContentType,
        Content = resultCard.Content,
        Preview = preview
    };

    attachments.Add(attachment);
}

return new MessagingExtensionResponse
{
    ComposeExtension = new MessagingExtensionResult
    {
        Type = "result",
        AttachmentLayout = "list",
        Attachments = attachments
    }
};
}
```
Note that you will implement `SearchProductsByCustomer` in Step 4.

# Step 3 - Update the command routing
In this step you will route the `companySearch` command to the handler you implemented in the previous step.

1. Open **searchBot.cs** and add the following. 

2. Underneath this statement:
```csharp
    case ProductSearchCommand.CommandId:
        return await ProductSearchCommand.HandleTeamsMessagingExtensionQueryAsync(turnContext, query, _configuration, cancellationToken);
```
Add this statement:
```csharp
    case CustomerSearchCommand.CommandId:
        return await CustomerSearchCommand.HandleTeamsMessagingExtensionQueryAsync(turnContext, query, _configuration, cancellationToken);
```
```text
Note that in the UI-based operation of the Message Extension / plugin, this command is explicitly called. However, when invoked by Microsoft 365 Copilot, the command is triggered by the Copilot orchestrator.
```
# Step 4 - Implement Product Search by Company
 You will implement a product search by Company name and return a list of the company's ordered products. Find this information using the tables below:

| Table         | Find        | Look Up By    |
| ------------- | ----------- | ------------- |
| Customer      | Customer Id | Customer Name |
| Orders        | Order Id    | Customer Id   |
| OrderDetail | Product       | Order Id      |

Here's how it works: 
Use the Customer table to find the Customer Id with the Customer Name. Query the Orders table with the Customer Id to retrieve the associated Order Ids. For each Order Id, find the associated products in the OrderDetail table. Finally, return a list of products ordered by the specified company name.

1. Open **.\NorthwindDB\Products.cs**

2. Add the new method `SearchProductsByCustomer()`

add the method:
```csharp
public async Task<List<IProductEx>> SearchProductsByCustomer(string companyName)
{
    var result = await GetAllProductsEx();

    var customers = await LoadReferenceData<Customer>(TABLE_NAME.CUSTOMER);
    string customerId = null;

    foreach (var customer in customers)
    {
        if (customer.CompanyName.IndexOf(companyName, StringComparison.OrdinalIgnoreCase) >= 0)
        {
            customerId = customer.CustomerID;
            break;
        }
    }

    if (string.IsNullOrEmpty(customerId))
    {
        return new List<IProductEx>();
    }

    var orders = await LoadReferenceData<Order>(TABLE_NAME.ORDER);
    var orderDetails = await LoadReferenceData<OrderDetail>(TABLE_NAME.ORDER_DETAIL);

    // Build an array of orders by customer id
    var customerOrders = orders.Where(o => o.CustomerID == customerId).ToList();

    // Build an array of order details for customer orders
    var customerOrderDetails = orderDetails
        .Where(od => customerOrders.Any(co => co.OrderID == od.OrderID))
        .ToList();

    // Filter products by the ProductID in the customerOrderDetails array
    result = result
        .Where(product => customerOrderDetails.Any(order => order.ProductID == product.ProductID))
        .ToList();

    return result;
}
```
# Step 5 - Run the App! Search for product by company name

Now you're ready to test the sample as a plugin for Microsoft 365 Copilot.

1. Delete the 'Northwest Inventory' app in Teams. This step is necessary since you are updating the manifest. Manifest updates require the app to be reinstalled. The cleanest way to do this is to first delete it in Teams.

    a. In the Teams sidebar, click on the three dots (...) 1️⃣. You should see Northwind Inventory 2️⃣ in the list of applications.

    b. Right click on the 'Northwest Inventory' icon and select uninstall 3️⃣.

    ![How to uninstall Northwind Inventory](./images/03-01-Uninstall-App.png)

2. Like you did in [Exercise 2 - Run the sample as a Copilot plugin](./Exercise%2003%20-%20Run%20in%20Copilot.md), start the app in Visual Studio using the **Microsoft Teams (browser)** profile.

3. In Teams click on **Chat** and then **Copilot**. Copilot should be the top-most option.
4. Click on the **Plugin icon** and select **Northwind Inventory** to enable the plugin.
5. Enter the prompt: 
```
What are the products ordered by 'Consolidated Holdings' in Northwind Inventory?
```

Here's the output in Copilot:
![03-07-response-customer-search](./images/03-07-response-customer-search.png)

Here are other prompts to try:
```
What are the products ordered by 'Consolidated Holdings' in Northwind Inventory? Please list the product name, price and supplier in a table.
```

Of course, you can test this new command also by using the sample as a Message Extension, like we did in [Exercise 2](./Exercise%2002%20-%20Run%20sample%20app.md). 

1. In the Teams sidebar, move to the **Chats** section and pick any chat or start a new chat with a colleague.
2. Click on the + sign to access to the Apps section.
3. Pick the Northwind Inventory app.
4. Notice how now you can see a new tab called **Customer**.
5. Search for **Consolidated Holdings** and see the products ordered by this company. They will match the ones that Copilot returned you in the previous step.

![The new command used as a message extension](./images/03-08-customer-message-extension.png)

## Congratulations
You've completed the exercise! Please proceed to [Exercise 5](./Exercise%2005%20-%20Code%20tour.md), in which you will explore the plugin source code and adaptive cards.

