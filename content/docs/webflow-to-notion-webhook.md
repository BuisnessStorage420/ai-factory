---
title: "Webflow to Notion Webhook Setup Guide"
date: 2026-06-17
lastmod: 2026-06-17
publishdate: 2026-06-17
draft: false
---

# How to Sync Webflow with Notion via Webhooks

This guide outlines the process of establishing a robust, one-way data synchronization from Webflow CMS to a Notion database using raw Webhooks and a custom intermediary service. This approach provides maximum control over data transformation and synchronization logic.

## 1. Prerequisites (Authentication, Keys, API requirements)

Before configuring any synchronization, ensure you have access to and understand the following components:

*   **Webflow Project with CMS:**
    *   A live Webflow project containing the CMS Collection you intend to synchronize.
    *   Permissions to access Project Settings > Integrations for Webhook setup.
    *   Familiarity with the structure and field types of your target Webflow CMS Collection.

*   **Notion Workspace and Integration:**
    *   A Notion workspace where you have full editing privileges.
    *   **Notion Integration:** Create a new integration at `https://www.notion.so/my-integrations`. Name it appropriately (e.g., "Webflow Sync"). Note down the **Internal Integration Token (API Key)**.
    *   **Notion Database:** Create a new Notion database (or identify an existing one) that will store the synchronized Webflow CMS items. Ensure the database has properties (columns) that map logically to your Webflow CMS fields (e.g., "Text" in Webflow -> "Text" in Notion, "Image URL" in Webflow -> "URL" in Notion). Share this database with your newly created Notion integration by clicking "..." next to the database title > "Add connections" > your integration name.
    *   **Notion Database ID:** Obtain the Database ID from the URL of your Notion database. It's the string of characters after `notion.so/` and before the `?` or before the database title. Example: `https://www.notion.so/YOUR_DATABASE_ID?v=...`

*   **Webhook Intermediary Service:**
    *   A serverless function (e.g., AWS Lambda, Google Cloud Function, Azure Function) or a custom backend application capable of:
        *   Receiving HTTP `POST` requests from Webflow.
        *   Parsing JSON payloads.
        *   Making authenticated HTTP `POST`/`PATCH` requests to the Notion API.
        *   This service will act as the bridge, transforming Webflow's webhook payload into a Notion API-compatible request.
    *   You will need to deploy this service and obtain its public endpoint URL. This URL will be your "Webhook URL" in Webflow.

*   **API Versions:**
    *   Notion API: Ensure your requests specify a `Notion-Version` header, e.g., `2022-06-28`.

## 2. Setting up the Trigger in Webflow

Configure Webflow to send a webhook request whenever a relevant event occurs in your CMS Collection.

1.  **Navigate to Webflow Project Settings:** In your Webflow project, go to `Project Settings` > `Integrations`.
2.  **Add New Webhook:** Scroll down to the "Webhooks" section and click `+ Add Webhook`.
3.  **Configure Webhook Details:**
    *   **Trigger Type:** Select the event that should initiate the sync. Common choices include:
        *   `Item Created`: Triggered when a new item is published in a Collection.
        *   `Item Updated`: Triggered when an existing item is published with changes.
        *   `Item Deleted`: Triggered when an item is deleted from a Collection.
        *   Select the specific CMS Collection for which you want to enable the webhook.
    *   **Webhook URL:** Enter the public endpoint URL of your intermediary service (from section 1).
    *   **Method:** Set to `POST`.
    *   **Name:** Give your webhook a descriptive name (e.g., "Notion Sync - Products").
4.  **Add Webhook:** Click "Add Webhook" to save your configuration.

**Webflow Webhook Payload Structure (Example for `Item Created`):**

When triggered, Webflow sends a JSON payload to your specified Webhook URL. For `Item Created` and `Item Updated` events, the most relevant data is within the `current` object.

```json
{
    "triggerType": "collection_item_created",
    "payload": {
        "_id": "651f1a6c4c3b5d2e0a1b2c3d", // Webflow Item ID
        "name": "My New Product",
        "slug": "my-new-product",
        "updatedOn": "2023-10-06T10:00:00.000Z",
        "createdOn": "2023-10-06T10:00:00.000Z",
        "isPublished": true,
        "isArchived": false,
        "isDraft": false,
        "current": {
            "_cid": "651f1a6c4c3b5d2e0a1b2c3e", // Collection ID
            "_id": "651f1a6c4c3b5d2e0a1b2c3d",
            "name": "My New Product", // This is the 'Name' field of the CMS item
            "product-description": "<p>Detailed description of the product.</p>", // Rich text field
            "product-price": 99.99, // Number field
            "product-image": { // Image field
                "fileId": "...",
                "url": "https://assets-global.website-files.com/...",
                "alt": "Product Image"
            },
            "category": "651f1a6c4c3b5d2e0a1b2c3f", // Reference field ID
            "item-url": "https://www.yourdomain.com/products/my-new-product", // System field
            "updated-on": "2023-10-06T10:00:00.000Z",
            "created-on": "2023-10-06T10:00:00.000Z",
            "published-on": "2023-10-06T10:00:00.000Z",
            "_archived": false,
            "_draft": false
        },
        "previous": null // Only present for 'Item Updated'
    }
}
```

## 3. Webhook Payload & Endpoint Configuration for Notion

This section details the logic for your intermediary service, which receives the Webflow webhook, processes its payload, and makes the appropriate calls to the Notion API.

### Intermediary Service Logic

Your intermediary service should perform the following steps upon receiving a Webflow webhook `POST` request:

1.  **Receive Payload:** Accept the incoming JSON payload from Webflow.
2.  **Parse Payload:** Extract the `triggerType` and the `payload.current` object (for `Item Created`/`Updated`) or `payload._id` (for `Item Deleted`).
3.  **Map Webflow to Notion Properties:** Transform the Webflow CMS item data into a format compatible with Notion's API. This involves:
    *   **Identifying Notion Properties:** Determine which Notion database properties correspond to which Webflow CMS fields.
    *   **Data Type Conversion:** Convert data types as necessary (e.g., Webflow's rich text HTML might need stripping or specific formatting for Notion's rich\_text property).
    *   **Handling Unique IDs:** For subsequent updates, it's crucial to store the Webflow Item ID (`payload._id`) in a dedicated Notion database property (e.g., a "Webflow ID" text property). This allows your intermediary service to query Notion, find the existing Notion page using the Webflow ID, and then `PATCH` it instead of creating a duplicate.

### Notion API Request Example (Create Page)

To create a new page (equivalent to a row) in your Notion database, your intermediary service will make an HTTP `POST` request to `https://api.notion.com/v1/pages`.

**Headers:**

```
Authorization: Bearer YOUR_NOTION_SECRET
Notion-Version: 2022-06-28
Content-Type: application/json
```

**Request Body (JSON Example for a "Product" database):**

This example assumes you have a Notion database with properties: "Name" (title), "Description" (rich\_text), "Price" (number), "Image URL" (url), and "Webflow ID" (rich\_text).

```json
{
    "parent": {
        "database_id": "YOUR_NOTION_DATABASE_ID"
    },
    "properties": {
        "Name": { // Maps to Webflow 'name'
            "title": [
                {
                    "text": {
                        "content": "Value from Webflow 'name' field"
                    }
                }
            ]
        },
        "Description": { // Maps to Webflow 'product-description'
            "rich_text": [
                {
                    "text": {
                        "content": "Value from Webflow 'product-description' field (e.g., HTML stripped, plain text)"
                    }
                }
            ]
        },
        "Price": { // Maps to Webflow 'product-price'
            "number": 99.99 // Value from Webflow 'product-price' field
        },
        "Image URL": { // Maps to Webflow 'product-image.url'
            "url": "https://assets-global.website-files.com/..." // Value from Webflow 'product-image.url'
        },
        "Webflow ID": { // Crucial for future updates
            "rich_text": [
                {
                    "text": {
                        "content": "651f1a6c4c3b5d2e0a1b2c3d" // Value from Webflow 'payload._id'
                    }
                }
            ]
        }
        // Add more properties as needed, mapping your Webflow fields
    }
}
```

**Handling Updates (`Item Updated` trigger):**

1.  When `triggerType` is `collection_item_updated`, your service should first query the Notion database using the Webflow Item ID (`payload._id`) stored in your "Webflow ID" property.
    *   Notion API Endpoint for querying: `POST https://api.notion.com/v1/databases/YOUR_NOTION_DATABASE_ID/query`
    *   Query body example: `{"filter": {"property": "Webflow ID", "rich_text": {"equals": "651f1a6c4c3b5d2e0a1b2c3d"}}}`
2.  If a matching Notion page is found (it will return `results` with `id` of the page), extract its `page_id`.
3.  Then, make an HTTP `PATCH` request to `https://api.notion.com/v1/pages/{page_id}` with an updated `properties` object. The `PATCH` request body only needs to include the properties that are changing.

**Handling Deletes (`Item Deleted` trigger):**

1.  When `triggerType` is `collection_item_deleted`, extract `payload._id`.
2.  Query Notion for the page with the corresponding "Webflow ID".
3.  If found, make an HTTP `PATCH` request to `https://api.notion.com/v1/pages/{page_id}` with a body `{"archived": true}` to archive the Notion page. Notion's API does not support direct deletion of pages.

## 4. Testing and Validating the Data Sync

Thorough testing is critical to ensure data integrity and reliable synchronization.

1.  **Deploy Intermediary Service:** Ensure your custom intermediary service is deployed and accessible at the Webhook URL configured in Webflow.
2.  **Trigger Webflow Event:**
    *   In your Webflow Designer, navigate to the CMS Collection for which the webhook is set up.
    *   **For `Item Created`:** Create a new item, fill in all relevant fields, and publish it.
    *   **For `Item Updated`:** Modify an existing item's content, then publish the changes.
    *   **For `Item Deleted`:** Delete an existing item and confirm the deletion.
3.  **Monitor Intermediary Service Logs:** Immediately after triggering, check the logs of your intermediary service.
    *   Confirm that the Webflow webhook request was received.
    *   Verify that the payload was parsed correctly.
    *   Look for any errors during the Notion API call (e.g., authentication failures, invalid property types, network issues).
4.  **Validate in Notion:**
    *   Open your target Notion database.
    *   **For `Item Created`:** A new row should appear with all the Webflow data accurately mapped to the Notion properties.
    *   **For `Item Updated`:** The corresponding row in Notion should show the updated values.
    *   **For `Item Deleted`:** The corresponding row in Notion should be archived (moved to the "Archived" section of your database or appear as "Deleted" if viewing all pages).
5.  **Troubleshooting:**
    *   **Webflow Webhook Logs:** In Webflow Project Settings > Integrations > Webhooks, click on your configured webhook. You can view the delivery status and payloads of recent webhook attempts. Look for "Success" or "Failed" statuses.
    *   **Notion Integration Permissions:** Double-check that your Notion integration has access to the specific database.
    *   **API Key/Database ID:** Verify that the Notion API key and database ID are correctly configured in your intermediary service.
    *   **Property Mapping:** Ensure the Notion property names and types in your JSON request body exactly match those in your Notion database. Mismatches are a common source of errors.
    *   **Data Types:** Be mindful of Notion's strict data type requirements (e.g., a "Number" property must receive a numeric value, not a string).
    *   **Test with `curl`:** If your intermediary service endpoint is publicly accessible, you can simulate a Webflow webhook payload using `curl` to test its processing logic directly, without needing to publish in Webflow every time.