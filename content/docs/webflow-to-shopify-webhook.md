---
title: "Webflow to Shopify Webhook Setup Guide"
date: 2026-06-17
lastmod: 2026-06-17
publishdate: 2026-06-17
draft: false
---

This guide details the process of synchronizing data between Webflow and Shopify using raw webhooks and a custom intermediary service. This method provides granular control over data transformation and synchronization logic, essential for complex integration requirements.

## 1. Prerequisites (Authentication, Keys, API requirements)

Before establishing the webhook-driven synchronization, ensure the following foundational elements are in place:

*   **Webflow Account:** Access to your Webflow project with permissions to modify CMS collections and configure webhooks.
*   **Shopify Store:** An active Shopify store with administrative access.
*   **Shopify Custom App:** Create a Custom App within your Shopify store (Admin > Settings > Apps and sales channels > Develop apps > Create an app). This app will generate:
    *   **Admin API Access Token:** A permanent token required to authenticate requests to the Shopify Admin API. Store this securely.
    *   **API Key and API Secret Key:** For specific authentication flows, though the Admin API Access Token is primary for direct API calls.
    *   **API Scopes:** Grant necessary permissions. For product synchronization, ensure at least `write_products` and `read_products` access.
*   **Intermediary Service:** A custom application or serverless function (e.g., AWS Lambda, Google Cloud Functions, Vercel Functions, Node.js/Python server) to act as the webhook receiver and data transformer. This service will:
    *   Expose an HTTP POST endpoint to receive webhooks from Webflow.
    *   Parse the incoming Webflow payload.
    *   Transform the data into a Shopify-compliant payload.
    *   Authenticate and make API calls to Shopify.
    *   Handle error logging and potentially retry logic.
*   **Development Environment:** Tools for coding your intermediary service (e.g., VS Code), Postman or similar for API testing, and a local tunnel tool like `ngrok` for exposing local development endpoints if necessary.
*   **Basic Knowledge:** Familiarity with JSON, HTTP methods (POST, PUT), and REST API concepts.

## 2. Setting up the Trigger in Webflow

Configure Webflow to dispatch a webhook whenever a relevant event occurs. We will focus on synchronizing Webflow CMS items as Shopify products.

1.  **Navigate to Project Settings:** In your Webflow Designer, go to **Project Settings** for the site you wish to integrate.
2.  **Access Integrations:** Click on the **Integrations** tab.
3.  **Add New Webhook:** Scroll down to the "Webhooks" section and click **"Add new webhook"**.
4.  **Configure Webhook Details:**
    *   **Name:** Provide a descriptive name (e.g., "Shopify Product Sync").
    *   **Trigger Type:** Select the event that initiates the sync. For new products or updates, `Collection item published` is highly recommended as it signifies a production-ready item. Other options include `Collection item created` or `Collection item updated` if draft syncing is desired.
    *   **Collection:** Specify the CMS Collection that contains the product data you want to sync (e.g., "Products").
    *   **Webhook URL:** This is the URL of the HTTP POST endpoint exposed by your intermediary service. For local development, an `ngrok` URL can be used; for production, a publicly accessible URL is required.

    *Example Webflow Webhook Configuration:*
    ```
    Name: Shopify Product Sync
    Trigger Type: Collection item published
    Collection: Products
    Webhook URL: https://your-intermediary-service.com/webflow-shopify-sync
    ```
5.  **Save Webhook:** Click "Add webhook" to activate it.

Webflow will now send a POST request to your specified `Webhook URL` whenever an item in the designated CMS collection is published. The payload will contain detailed information about the published item.

*Example Webflow `collection_item_published` Payload Structure (simplified):*
```json
{
  "triggerId": "651a3d90f23d4e0046b0e8b2",
  "triggerType": "collection_item_published",
  "site": {
    "id": "651a3d90f23d4e0046b0e8b0",
    "name": "My Webflow Store"
  },
  "collection": {
    "id": "651a3d90f23d4e0046b0e8b1",
    "name": "Products",
    "slug": "products"
  },
  "item": {
    "_id": "651a3d90f23d4e0046b0e8b3",
    "name": "Webflow Product Title",
    "slug": "webflow-product-title",
    "price": 49.99,
    "description": "<p>This is a product description from Webflow.</p>",
    "main-image": {
      "fileId": "651a3d90f23d4e0046b0e8b4",
      "url": "https://uploads-ssl.webflow.com/651a3d90f23d4e0046b0e8b0/651a3d90f23d4e0046b0e8b4_product-image.jpg",
      "alt": "Product Main Image"
    },
    "category": {
      "_id": "651a3d90f23d4e0046b0e8b5",
      "name": "Electronics",
      "slug": "electronics"
    },
    "published-on": "2023-10-27T10:00:00.000Z",
    "updated-on": "2023-10-27T10:05:00.000Z"
  }
}
```

## 3. Webhook Payload & Endpoint Configuration for Shopify

This section details the critical intermediary service logic for receiving the Webflow webhook, transforming the data, and interacting with the Shopify Admin API.

**Intermediary Service Logic:**

1.  **Receive Webflow Webhook:**
    *   Your intermediary service must listen for `POST` requests at the configured `Webhook URL`.
    *   Parse the incoming request body as JSON.
2.  **Authenticate Request (Optional but Recommended):**
    *   Webflow provides an optional `X-Webflow-Signature` header for verifying the webhook's origin. Implement signature verification using your Webflow webhook secret (found in Webflow project settings) for enhanced security.
3.  **Extract Data:**
    *   Access the relevant fields from the `item` object within the Webflow payload. Key fields typically include `name`, `slug`, `description`, `price`, `main-image.url`, and the `_id` for unique identification.
4.  **Determine Action (Create or Update):**
    *   To handle updates, your intermediary service needs to determine if the Webflow item already exists as a product in Shopify.
    *   **Strategy:** Utilize the Webflow `item._id` as a unique identifier. You can store this ID in a Shopify product's `SKU` field or, more robustly, as a `metafield` during initial product creation.
    *   Before creating a product, query Shopify's API (`GET /admin/api/2023-10/products.json?fields=id,metafields`) using the Webflow `item._id` to check for an existing product.
5.  **Transform Data to Shopify Product API Format:**
    *   Map Webflow fields to their corresponding Shopify Product API fields. Note that Shopify expects prices as strings and may require specific image formats.
    *   Consider default values for fields not present in Webflow (e.g., `vendor`, `product_type`, `status`).

    *Example Shopify Product `POST` Payload (for creation):*
    ```json
    {
      "product": {
        "title": "Webflow Product Title",
        "body_html": "<p>This is a product description from Webflow.</p>",
        "vendor": "Your Brand Name",
        "product_type": "General",
        "status": "active",
        "tags": "Webflow-Sync",
        "variants": [
          {
            "price": "49.99",
            "sku": "WEBFLOW_ITEM_ID_651a3d90f23d4e0046b0e8b3", // Use Webflow _id for SKU
            "inventory_management": null, // Shopify manages inventory
            "inventory_policy": "deny",   // Don't allow overselling
            "fulfillment_service": "manual"
          }
        ],
        "images": [
          {
            "src": "https://uploads-ssl.webflow.com/651a3d90f23d4e0046b0e8b0/651a3d90f23d4e0046b0e8b4_product-image.jpg",
            "alt": "Product Main Image"
          }
        ],
        "metafields": [ // Store Webflow item ID for future lookups
            {
                "key": "webflow_item_id",
                "namespace": "custom",
                "value": "651a3d90f23d4e0046b0e8b3",
                "type": "single_line_text_field"
            }
        ]
      }
    }
    ```
6.  **Make Shopify API Call:**
    *   **Authentication:** Include the `X-Shopify-Access-Token` header with your Admin API Access Token.
    *   **Endpoint:**
        *   **Create:** `POST https://{your-store-name}.myshopify.com/admin/api/2023-10/products.json`
        *   **Update:** `PUT https://{your-store-name}.myshopify.com/admin/api/2023-10/products/{shopify_product_id}.json` (where `shopify_product_id` is retrieved from your lookup).
    *   **Headers:**
        ```
        Content-Type: application/json
        X-Shopify-Access-Token: shpat_YOUR_ADMIN_API_ACCESS_TOKEN
        ```
7.  **Handle API Response & Error Logging:**
    *   Log successful creations/updates.
    *   Implement robust error handling for Shopify API responses (e.g., status codes 4xx, 5xx) and log details for debugging. Consider exponential backoff for rate limit errors.

## 4. Testing and Validating the Data Sync

Thorough testing is crucial to ensure reliable data synchronization.

1.  **Initial Setup Test:**
    *   Deploy your intermediary service to a publicly accessible URL (or use `ngrok` for local testing).
    *   Configure the Webflow webhook with this URL.
    *   Use a tool like [Webhook.site](https://webhook.site) as your `Webhook URL` initially to confirm Webflow is sending payloads correctly and to inspect their structure.
2.  **Trigger Webflow Event:**
    *   In your Webflow Designer, create a new CMS item in the designated collection and **publish** it.
    *   For existing items, make a significant change and **publish** it.
3.  **Monitor Intermediary Service:**
    *   Observe the logs of your intermediary service. Verify that it received the webhook, parsed the data, and successfully made an API call to Shopify. Look for any error messages or warnings.
4.  **Verify in Shopify:**
    *   Log into your Shopify Admin panel.
    *   Navigate to **Products**.
    *   Confirm that the new product has been created or the existing product has been updated with the correct data (title, description, price, images, tags, etc.).
    *   Check for the custom `metafield` containing the Webflow item ID if implemented.
5.  **Test Update Logic:**
    *   Modify an existing synchronized product in Webflow (e.g., change its name or price) and publish the changes.
    *   Verify that the corresponding product in Shopify is updated correctly, not duplicated.
6.  **Edge Case Testing:**
    *   **Missing Fields:** Test what happens if a critical field (e.g., price) is missing in Webflow. Your intermediary service should handle this gracefully (e.g., provide a default, throw an error, or skip synchronization).
    *   **Image Issues:** Test with items that have no images or broken image URLs.
    *   **Rate Limits:** While harder to simulate, be aware of Shopify's API rate limits and consider implementing retry logic with exponential backoff in your intermediary service for production readiness.
    *   **Error Handling:** Intentionally cause an error (e.g., use an invalid Shopify API token) to confirm your service logs errors effectively.

By following these steps, you can establish a robust, custom synchronization pipeline between Webflow and Shopify, leveraging the power of raw webhooks for precise data control.