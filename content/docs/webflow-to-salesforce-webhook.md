---
title: "Webflow to Salesforce Webhook Setup Guide"
date: 2026-06-17
lastmod: 2026-06-17
publishdate: 2026-06-17
draft: false
---

As a principal integration engineer, this guide outlines the technical steps to synchronize data directly between Webflow and Salesforce using raw webhooks, leveraging Salesforce's Apex REST capabilities for robust, custom processing. This approach minimizes reliance on third-party integration platforms, offering granular control over data flow and transformation.

# How to Sync Webflow with Salesforce via Webhooks

## 1. Prerequisites (Authentication, Keys, API requirements)

Successful integration hinges on having the necessary access and understanding of both platforms.

### Webflow Requirements
*   **Webflow Site Access**: Full administrative access to your Webflow site, specifically to "Site Settings > Integrations > Webhooks."
*   **Webflow Forms or CMS Collections**: An existing Webflow form or a CMS Collection that will act as the data source for your synchronization trigger. You must be familiar with the field names (e.g., `Email`, `First Name`, `Product Name`) as they will appear in the webhook payload.
*   **Webhook Feature**: Ensure the Webflow plan supports webhooks (available on all paid plans).

### Salesforce Requirements
*   **Salesforce Org Access**: Access to a Salesforce Developer, Sandbox, or Production organization where the data will reside.
*   **Administrator Permissions**: A Salesforce user with "Customize Application" and "Author Apex" permissions to create and deploy Apex classes and potentially custom objects or fields.
*   **API Enabled**: Ensure API access is enabled for the Salesforce user profile that will be involved in Apex code deployment and execution.
*   **My Domain**: A "My Domain" configured in your Salesforce org is highly recommended for stable and custom Salesforce URLs, which are crucial for exposed REST endpoints.
*   **Security Token (if applicable)**: While not directly used by the Apex REST endpoint for authentication *from Webflow*, if your Apex code interacts with other Salesforce APIs using traditional username/password authentication, a security token might be required alongside the password. For this direct webhook scenario, the Apex endpoint itself runs within the Salesforce platform's security context.

### Endpoint Handler Requirements (Salesforce Apex REST)
For "raw" webhooks, an external middleware is often bypassed by having Salesforce directly expose a REST endpoint via Apex. This requires:
*   **Apex Development Environment**: Access to Salesforce Developer Console or a suitable IDE (like VS Code with Salesforce extensions) for Apex class creation and deployment.
*   **Understanding of Apex REST**: Familiarity with `@RestResource` annotations and handling HTTP methods like `POST` within Apex. This Apex class will receive the raw JSON payload, parse it, and perform DML operations in Salesforce.

## 2. Setting up the Trigger in Webflow

Configuring the webhook in Webflow is straightforward and defines what event triggers the data dispatch and where it should be sent.

1.  **Navigate to Webflow Site Settings**:
    *   Open your Webflow project in the Designer.
    *   Go to `Site Settings` (accessible from the left panel, usually a gear icon).
    *   Click on the `Integrations` tab.
    *   Scroll down to the `Webhooks` section.

2.  **Add New Webhook**:
    *   Click the `Add Webhook` button.
    *   **Trigger Type**: Select the event that should initiate the data transfer. Common choices include:
        *   `Form Submission`: Ideal for lead capture, contact forms, or newsletter sign-ups.
        *   `CMS Item Created`: Useful for syncing new blog posts, products, or team members to Salesforce (e.g., as custom objects).
        *   `CMS Item Updated`: For keeping Salesforce records synchronized with changes in Webflow CMS content.
    *   **Webhook URL**: This is the critical piece. It must be the public-facing URL of your Salesforce Apex REST endpoint. A typical Salesforce Apex REST endpoint URL follows this structure:
        `https://<your-my-domain>.my.salesforce.com/services/apexrest/<your-url-mapping>`
        For example: `https://acmecorp.my.salesforce.com/services/apexrest/v1/webflow/data`
        *Note*: Replace `<your-my-domain>` with your actual Salesforce My Domain name and `<your-url-mapping>` with the URL mapping defined in your Apex class (as shown in Section 3).
    *   **HTTP Method**: Ensure `POST` is selected. Webflow webhooks exclusively use `POST` to send data.

3.  **Save Webhook**: Click `Add Webhook` to save your configuration. Webflow will now send a JSON payload to the specified URL whenever the selected trigger event occurs.

### Example Webflow Form Submission Payload
When a form is submitted, Webflow dispatches a JSON payload similar to this:

```json
{
  "name": "Form Submission",
  "site": {
    "id": "651a1b2c3d4e5f6a7b8c9d0e",
    "name": "My Company Website"
  },
  "formId": "651a1b2c3d4e5f6a7b8c9d1f",
  "data": {
    "name": "Contact Form",
    "Email": "jane.doe@example.com",
    "First Name": "Jane",
    "Last Name": "Doe",
    "Company": "Innovate Corp",
    "Message": "I'm interested in a demo."
  },
  "orderedItems": [],
  "fieldData": {}
}
```

## 3. Webhook Payload & Endpoint Configuration for Salesforce

The core of this integration lies in creating a Salesforce Apex REST endpoint to receive, parse, and act upon the Webflow webhook payload.

### Salesforce Apex REST Endpoint
Create an Apex class that exposes a `POST` method via the `@RestResource` annotation. This class will parse the incoming JSON and perform DML operations (e.g., `upsert` a Lead or Contact).

```apex
@RestResource(urlMapping='/v1/webflow/data') // Matches the /services/apexrest/v1/webflow/data part of your Webhook URL
global with sharing class WebflowWebhookReceiver {

    @HttpPost
    global static void handleWebhook() {
        RestRequest req = RestContext.request;
        RestResponse res = RestContext.response;
        res.addHeader('Content-Type', 'application/json'); // Ensure JSON response

        try {
            // Get the raw JSON payload from Webflow
            String jsonPayload = req.requestBody.toString();
            System.debug('Webflow Webhook Payload: ' + jsonPayload);

            // Deserialize the JSON payload. Using untyped deserialization for flexibility.
            Map<String, Object> payloadMap = (Map<String, Object>) JSON.deserializeUntyped(jsonPayload);

            // Identify the trigger type (e.g., "Form Submission" or "Collection Item Created")
            String eventName = (String)payloadMap.get('name');

            if ('Form Submission'.equals(eventName)) {
                processFormSubmission(payloadMap, res);
            } else if ('Collection Item Created'.equals(eventName) || 'Collection Item Updated'.equals(eventName)) {
                processCmsItem(payloadMap, res);
            } else {
                res.statusCode = 400;
                res.responseBody = '{"status":"error", "message":"Unsupported event type."}';
            }

        } catch (Exception e) {
            System.debug('Error processing Webflow webhook: ' + e.getMessage() + ' at line ' + e.getLineNumber() + '\nStack Trace: ' + e.getStackTraceString());
            res.statusCode = 500;
            res.responseBody = '{"status":"error", "message":"An internal server error occurred: ' + e.getMessage() + '"}';
            // Consider logging to a custom object or sending email alerts for critical errors
        }
    }

    private static void processFormSubmission(Map<String, Object> payloadMap, RestResponse res) {
        Map<String, Object> formData = (Map<String, Object>) payloadMap.get('data');

        String email = (String) formData.get('Email');
        String firstName = (String) formData.get('First Name');
        String lastName = (String) formData.get('Last Name');
        String company = (String) formData.get('Company');

        if (String.isBlank(email)) {
            res.statusCode = 400;
            res.responseBody = '{"status":"error", "message":"Email is a required field for form submissions."}';
            return;
        }

        Lead newOrExistingLead;
        try {
            // Check for existing Lead by Email
            List<Lead> existingLeads = [SELECT Id, FirstName, LastName, Company, Email FROM Lead WHERE Email = :email LIMIT 1];

            if (!existingLeads.isEmpty()) {
                newOrExistingLead = existingLeads[0];
                System.debug('Updating existing Lead: ' + newOrExistingLead.Id);
            } else {
                newOrExistingLead = new Lead();
                newOrExistingLead.Status = 'New';
                newOrExistingLead.LeadSource = 'Webflow Form';
                System.debug('Creating new Lead.');
            }

            // Map Webflow form fields to Salesforce Lead fields
            newOrExistingLead.Email = email;
            if (firstName != null) newOrExistingLead.FirstName = firstName;
            if (lastName != null) newOrExistingLead.LastName = lastName;
            if (company != null) newOrExistingLead.Company = company;

            upsert newOrExistingLead Email; // Upsert using Email as the external ID

            res.statusCode = 200;
            res.responseBody = '{"status":"success", "message":"Lead processed successfully."}';

        } catch (DmlException dmlEx) {
            System.debug('DML Error during Lead upsert: ' + dmlEx.getMessage());
            res.statusCode = 400; // Indicate a client-side (data) error
            res.responseBody = '{"status":"error", "message":"Data processing error: ' + dmlEx.getDmlMessage(0) + '"}';
        }
    }

    private static void processCmsItem(Map<String, Object> payloadMap, RestResponse res) {
        Map<String, Object> itemData = (Map<String, Object>) payloadMap.get('item');

        String itemId = (String) itemData.get('_id'); // Webflow CMS Item ID
        String itemName = (String) itemData.get('name'); // e.g., Blog Post Title
        String itemSlug = (String) itemData.get('slug'); // e.g., URL slug
        // Add more CMS fields as needed, e.g., 'post-body', 'published-on'

        if (String.isBlank(itemId)) {
            res.statusCode = 400;
            res.responseBody = '{"status":"error", "message":"CMS Item ID is required."}';
            return;
        }

        // Example: Upserting to a custom object named 'Webflow_CMS_Item__c'
        // Ensure you have an External ID field (e.g., Webflow_Item_ID__c) on your custom object
        Webflow_CMS_Item__c cmsItem;
        try {
            List<Webflow_CMS_Item__c> existingItems = [SELECT Id, Name, Webflow_Item_ID__c FROM Webflow_CMS_Item__c WHERE Webflow_Item_ID__c = :itemId LIMIT 1];

            if (!existingItems.isEmpty()) {
                cmsItem = existingItems[0];
                System.debug('Updating existing CMS Item: ' + cmsItem.Id);
            } else {
                cmsItem = new Webflow_CMS_Item__c();
                cmsItem.Webflow_Item_ID__c = itemId; // Map Webflow ID to External ID field
                System.debug('Creating new CMS Item.');
            }

            cmsItem.Name = itemName;
            cmsItem.Slug__c = itemSlug; // Assuming a custom field Slug__c
            // Map other fields like cmsItem.Body__c = (String) itemData.get('post-body');

            upsert cmsItem Webflow_Item_ID__c; // Upsert using the external ID field

            res.statusCode = 200;
            res.responseBody = '{"status":"success", "message":"CMS Item processed successfully."}';

        } catch (DmlException dmlEx) {
            System.debug('DML Error during CMS Item upsert: ' + dmlEx.getMessage());
            res.statusCode = 400;
            res.responseBody = '{"status":"error", "message":"Data processing error: ' + dmlEx.getDmlMessage(0) + '"}';
        }
    }
}

```

### Example Webflow CMS Item Created Payload
When a CMS item is created, the payload structure changes:

```json
{
  "name": "Collection Item Created",
  "site": {
    "id": "651a1b2c3d4e5f6a7b8c9d0e",
    "name": "My Company Website"
  },
  "collection": {
    "id": "651a1b2c3d4e5f6a7b8c9e0f",
    "name": "Blog Posts"
  },
  "item": {
    "_cid": "651a1b2c3d4e5f6a7b8c9e0f",
    "_id": "651a1b2c3d4e5f6a7b8c9f0e",
    "name": "The Future of Web Design",
    "slug": "the-future-of-web-design",
    "featured": false,
    "post-body": "<p>Content of the blog post...</p>",
    "published-on": "2023-10-27T14:30:00.000Z",
    "created-on": "2023-10-27T14:25:00.000Z",
    "_archived": false,
    "_draft": false
  }
}
```

## 4. Testing and Validating the Data Sync

Thorough testing is crucial to ensure data integrity and reliable synchronization.

1.  **Trigger the Webhook in Webflow**:
    *   **For Form Submissions**: Go to your live Webflow site and submit the form with sample data. Use distinct values (e.g., a unique email address) for easy identification in Salesforce.
    *   **For CMS Items**: In the Webflow Designer, create or update a CMS item in the collection configured for the webhook, then publish your site.

2.  **Verify Webflow Webhook Delivery**:
    *   Navigate back to `Site Settings > Integrations > Webhooks` in Webflow.
    *   Click on your configured webhook. You should see a log of recent deliveries.
    *   Look for a `Status: 200 OK` indicating Webflow successfully sent the payload to your Salesforce endpoint. If you see errors (e.g., `404 Not Found`, `500 Internal Server Error`), check your Webhook URL and the Apex class deployment.

3.  **Monitor Salesforce Developer Console (Debug Logs)**:
    *   Open the Salesforce Developer Console (`Setup > Developer Console`).
    *   Go to `Debug > Change Log Levels`. Add a `Finest` log level for the `WebflowWebhookReceiver` class.
    *   Trigger the webhook again.
    *   Observe the debug logs in the Developer Console. You should see the `System.debug` statements from your Apex class, confirming the payload receipt, deserialization, and DML operations. Look for any `DmlException` or other error messages.

4.  **Validate Data in Salesforce**:
    *   Navigate to the relevant Salesforce object (e.g., `Leads`, `Contacts`, `Webflow CMS Items__c`).
    *   Search for the record using the unique data you submitted (e.g., the email address from the form, the Webflow Item ID).
    *   Verify that:
        *   The record was created or updated correctly.
        *   All mapped fields contain the expected data from Webflow.
        *   Data types and formats are correct (e.g., dates, numbers).
        *   Any default values or automation (e.g., Lead Status) applied as expected.

5.  **Test Error Scenarios**:
    *   **Missing Required Fields**: Submit a form or create a CMS item missing a critical field (e.g., `Email` for a Lead). Verify that your Apex code correctly handles this and returns an appropriate error status (e.g., `400 Bad Request`) to Webflow.
    *   **Invalid Data**: If applicable, test with data that might violate Salesforce validation rules (e.g., text in a number field).
    *   **Edge Cases**: Consider long strings, special characters, or empty values for different fields.

By following these steps, you can establish a robust, direct, and custom data synchronization channel between Webflow and Salesforce using raw webhooks.