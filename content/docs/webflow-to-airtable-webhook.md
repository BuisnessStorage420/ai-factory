---
title: "Webflow to Airtable Webhook Setup Guide"
date: 2026-06-17
lastmod: 2026-06-17
publishdate: 2026-06-17
draft: false
---

# How to Sync Webflow with Airtable via Webhooks

This guide details the synchronization of data from Webflow to Airtable using raw Webhooks, bypassing intermediary no-code automation platforms for direct integration. This approach requires an custom-built webhook receiver endpoint to process Webflow payloads and interact with the Airtable API.

## 1. Prerequisites (Authentication, Keys, API requirements)

Successful integration necessitates specific credentials and a defined architectural component:

1.  **Webflow Site Access:**
    *   **Webflow Site ID:** Unique identifier for your Webflow site.
    *   **Webflow Collection ID:** Unique identifier for the specific Collection you intend to monitor.
    *   **Webflow API Access (Optional but recommended for full flexibility):** While not strictly required for sending webhooks, having a Webflow API key (`WEBFLOW_API_KEY`) is beneficial for programmatic interaction or debugging. Configure within Site Settings > Integrations > API.
    *   **Target Event:** Identify the Webflow event (e.g., `Collection item created`, `Collection item updated`, `Form submission`) that will trigger the webhook.

2.  **Airtable Base & Table Setup:**
    *   **Airtable Personal Access Token (PAT):** Generate a PAT with `data.records:write` and `schema.bases:read` permissions for your target base. This is the preferred authentication method over legacy API keys. Configure from your Airtable account settings. (`AIRTABLE_PAT`)
    *   **Airtable Base ID:** Unique identifier for your Airtable base. Found in the URL (`airtable.com/{baseId}/...`) or via the Airtable API documentation for your base. (`AIRTABLE_BASE_ID`)
    *   **Airtable Table Name/ID:** The name or ID of the specific table within your base where records will be synchronized. (`AIRTABLE_TABLE_NAME`)
    *   **Airtable Table Schema:** A clear understanding of your Airtable table's field names and data types (e.g., `Name` (Single line text), `Description` (Long text), `Status` (Single select)). Ensure these align with the data you expect from Webflow.

3.  **Webhook Receiver Endpoint:**
    *   **Custom Application/Serverless Function:** Airtable's API does not natively expose a direct public endpoint for arbitrary Webhook reception. You must deploy a custom application or serverless function (e.g., AWS Lambda, Google Cloud Functions, Azure Functions, Vercel Edge Functions, Netlify Functions) that will:
        *   Expose a public HTTP POST endpoint.
        *   Receive and parse the incoming JSON payload from Webflow.
        *   Construct and execute API requests to Airtable using the provided PAT, Base ID, and Table Name.
        *   Handle responses and errors, providing logging and potentially retry mechanisms.
    *   **Endpoint URL:** The fully qualified public URL for this receiver endpoint. (`YOUR_WEBHOOK_RECEIVER_URL`)

## 2. Setting up the Trigger in Webflow

Configure Webflow to dispatch a Webhook event to your custom endpoint when a specified action occurs:

1.  **Navigate to Webflow Site Settings:** From your Webflow Designer, go to **Site Settings**.
2.  **Access Integrations:** Click on the **Integrations** tab.
3.  **Locate Webhooks Section:** Scroll down to the **Webhooks** section.
4.  **Add New Webhook:** Click the **"Add Webhook"** button.
5.  **Configure Webhook Details:**
    *   **Trigger Type:** Select the event that should initiate the webhook. For synchronizing new collection items, choose `Collection item created`. For updates, select `Collection item updated`.
    *   **Collection:** If the trigger type is related to a Collection (e.g., `Collection item created`), select the specific Collection you intend to synchronize (e.g., `Products`).
    *   **Webhook URL:** Enter the full public URL of your custom webhook receiver endpoint (e.g., `https://api.yourdomain.com/webflow-airtable-sync`).
    *   **API Version:** Select `1.0.0` (or the latest stable version available).
6.  **Add Webhook:** Click the **"Add Webhook"** button to save the configuration.

## 3. Webhook Payload & Endpoint Configuration for Airtable

Upon trigger, Webflow sends a JSON payload to your `YOUR_WEBHOOK_RECEIVER_URL`. Your custom endpoint must parse this payload and construct an Airtable API request.

### Webflow Webhook Payload Example (Collection Item Created)

A typical payload for `Collection item created` will include metadata about the event and the newly created item's data:

```json
{
  "triggerType": "collection_item_created",
  "webhookId": "651f8a7e082103328e21a0a5",
  "data": {
    "site": {
      "_id": "651f87ae0342111d9d5c21f1",
      "name": "My E-commerce Site",
      "timezone": "America/New_York"
    },
    "collection": {
      "_id": "651f8a7d0342111d9d5c21f2",
      "name": "Products",
      "slug": "products"
    },
    "item": {
      "_id": "651f8f9e0342111d9d5c21f3",
      "_cid": "651f8a7d0342111d9d5c21f2",
      "name": "Premium Widget",
      "slug": "premium-widget",
      "product-description": "An advanced widget with superior features.",
      "price": 49.99,
      "sku": "PW-001",
      "_archived": false,
      "_draft": false,
      "_createdOn": "2023-10-06T14:30:00.000Z",
      "_updatedOn": "2023-10-06T14:30:00.000Z"
    }
  }
}
```

### Airtable Webhook Receiver Endpoint Logic (Conceptual Node.js Example)

Your serverless function or application must perform the following:

1.  **Receive POST Request:** Accept the incoming HTTP POST request.
2.  **Parse Payload:** Extract relevant data from `req.body.data.item`.
3.  **Map Fields:** Map Webflow item fields to your Airtable table's column names. Ensure data types are compatible.
4.  **Construct Airtable API Request:** Formulate a `POST` request to the Airtable API for record creation.

```javascript
// Example conceptual code for a serverless function (e.g., AWS Lambda, Vercel Function)
const AIRTABLE_PAT = process.env.AIRTABLE_PAT;
const AIRTABLE_BASE_ID = process.env.AIRTABLE_BASE_ID;
const AIRTABLE_TABLE_NAME = process.env.AIRTABLE_TABLE_NAME;

module.exports = async (req, res) => {
  if (req.method !== 'POST') {
    return res.status(405).send('Method Not Allowed');
  }

  const webflowPayload = req.body;

  if (!webflowPayload || !webflowPayload.data || !webflowPayload.data.item) {
    console.error('Invalid Webflow payload received:', webflowPayload);
    return res.status(400).send('Bad Request: Missing item data.');
  }

  const item = webflowPayload.data.item;

  // Map Webflow fields to Airtable fields
  const airtableRecordFields = {
    "Product Name": item.name,           // Map Webflow 'name' to Airtable 'Product Name'
    "Description": item['product-description'], // Map custom Webflow field to Airtable 'Description'
    "Price": item.price,                 // Map Webflow 'price' to Airtable 'Price'
    "SKU": item.sku,                     // Map Webflow 'sku' to Airtable 'SKU'
    "Webflow ID": item._id               // Store Webflow item ID for reference
  };

  try {
    const airtableApiUrl = `https://api.airtable.com/v0/${AIRTABLE_BASE_ID}/${AIRTABLE_TABLE_NAME}`;

    const airtableResponse = await fetch(airtableApiUrl, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${AIRTABLE_PAT}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        "records": [
          { "fields": airtableRecordFields }
        ]
      })
    });

    if (!airtableResponse.ok) {
      const errorData = await airtableResponse.json();
      console.error('Airtable API Error:', airtableResponse.status, errorData);
      return res.status(500).json({ error: 'Failed to create record in Airtable', details: errorData });
    }

    const createdRecords = await airtableResponse.json();
    console.log('Successfully created record in Airtable:', createdRecords);
    res.status(200).json({ message: 'Synchronization successful', records: createdRecords.records });

  } catch (error) {
    console.error('Error processing webhook:', error);
    res.status(500).json({ error: 'Internal Server Error', details: error.message });
  }
};
```

### Airtable API Request Body Example

The `body` for the `fetch` request to Airtable (for creating a new record) should conform to this structure:

```json
{
  "records": [
    {
      "fields": {
        "Product Name": "Premium Widget",
        "Description": "An advanced widget with superior features.",
        "Price": 49.99,
        "SKU": "PW-001",
        "Webflow ID": "651f8f9e0342111d9d5c21f3"
      }
    }
  ]
}
```

## 4. Testing and Validating the Data Sync

After configuration and deployment of your webhook receiver, meticulous testing is critical:

1.  **Trigger the Webflow Event:**
    *   In your Webflow Designer, navigate to the Collection you configured (e.g., `Products`).
    *   Create a new item within that Collection, ensuring all relevant fields are populated.
    *   Publish your Webflow site to ensure the changes are live and the webhook fires.

2.  **Monitor Webflow Webhook Logs:**
    *   Return to **Site Settings > Integrations > Webhooks** in Webflow.
    *   Inspect the "Logs" section for your newly created webhook. Look for a `200 OK` or `204 No Content` status, indicating Webflow successfully delivered the payload to your endpoint. A `5xx` status suggests an issue with your endpoint's accessibility or initial response.

3.  **Review Endpoint Logs:**
    *   Access the logs for your deployed serverless function or application.
    *   Verify that the Webflow payload was received, parsed correctly, and the Airtable API request was successfully initiated.
    *   Look for any errors reported during field mapping or the Airtable API call.

4.  **Verify Data in Airtable:**
    *   Open your target Airtable Base and Table.
    *   Confirm that a new record has been created, and that the data from Webflow has been accurately mapped to the respective Airtable fields.
    *   Check for any data type mismatches or truncation.

### Troubleshooting Common Issues:

*   **Webhook URL Inaccessibility:** Ensure `YOUR_WEBHOOK_RECEIVER_URL` is publicly accessible and accepts POST requests.
*   **Airtable API Key/PAT Permissions:** Confirm the Airtable Personal Access Token has `data.records:write` permissions for the target base.
*   **Field Mapping Discrepancies:** Mismatches between Webflow item field names (especially custom fields, which often use slugs) and Airtable column names are a frequent cause of failure. Ensure exact matches or correct transformations.
*   **Data Type Mismatches:** Sending a text string to a number field in Airtable, or an array to a single-select field without proper handling, will result in errors.
*   **Rate Limits:** Be aware of Airtable API rate limits (5 requests per second per base). Implement backoff and retry logic in your endpoint for production systems.
*   **Security:** For production systems, consider validating the Webflow signature in the `X-Webflow-Hmac-SHA256` header to ensure requests originate from Webflow.