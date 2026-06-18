---
title: "Webflow to Stripe Webhook Setup Guide"
date: 2026-06-17
lastmod: 2026-06-17
publishdate: 2026-06-17
draft: false
---

# How to Sync Webflow with Stripe via Webhooks

This guide outlines the process for synchronizing data between Webflow and Stripe using raw webhooks, focusing on a robust, practical, and server-side approach. As Webflow's native webhooks primarily send data and Stripe's API requires authenticated calls, an intermediary serverless function or custom middleware is essential for this integration.

## 1. Prerequisites (Authentication, Keys, API requirements)

Successful integration mandates access to specific credentials and a foundational understanding of both platforms.

*   **Webflow Account:**
    *   An active Webflow site with a hosting plan that supports webhooks (e.g., CMS, Business, or Ecommerce).
    *   Access to Project Settings within the Webflow Designer for configuring webhooks.
    *   (Optional, but recommended for advanced bi-directional sync): A Webflow API token if your intermediary service needs to update Webflow CMS items based on Stripe events. Generate this in Project Settings > Integrations > API Access.

*   **Stripe Account:**
    *   An active Stripe account with developer access.
    *   **Stripe Secret API Key:** Found in your Stripe Dashboard under Developers > API Keys. This key (`sk_live_...` or `sk_test_...`) is critical for authenticating server-side API requests from your intermediary service to Stripe. **Never expose this key client-side.**
    *   (Optional, but crucial for receiving Stripe webhooks): A Webhook Signing Secret, generated when you create a webhook endpoint in Stripe. This is used to verify the authenticity of incoming Stripe webhooks if you need Stripe to push data to *your* application. While this guide primarily focuses on Webflow *triggering* Stripe actions, a complete sync may involve this.

*   **Intermediary Service (Crucial):**
    *   A serverless function (e.g., AWS Lambda, Google Cloud Functions, Azure Functions, Cloudflare Workers) or a dedicated web server application (Node.js, Python Flask/Django, PHP Laravel, Ruby on Rails).
    *   This service will act as a bridge:
        1.  It will expose a public HTTP POST endpoint to receive Webflow webhooks.
        2.  It will parse the incoming Webflow payload.
        3.  It will then use the Stripe Secret API Key to make authenticated API calls to Stripe.
    *   **Tools:** Standard HTTP client libraries for your chosen language (e.g., `axios` or `node-fetch` for Node.js, `requests` for Python) and the official Stripe SDK for your chosen language (e.g., `stripe-node`, `stripe-python`).

## 2. Setting up the Trigger in Webflow

Configure Webflow to dispatch a webhook event to your intermediary service whenever a specific action occurs.

1.  **Navigate to Webflow Project Settings:** Open your Webflow project, go to "Project Settings" (icon in the left sidebar), then select the "Integrations" tab.
2.  **Access Webhooks:** Scroll down to the "Webhooks" section.
3.  **Add New Webhook:** Click the "Add New Webhook" button.
4.  **Configure Webhook Details:**
    *   **Trigger Type:** Select the event that will initiate the webhook. Common choices include:
        *   `Form Submission`: When a user submits a form on your Webflow site. (Most common for syncing user data or initiating payments).
        *   `CMS Item Created`, `CMS Item Updated`, `CMS Item Deleted`: For synchronizing product catalogs, user profiles stored in CMS, etc.
        *   `E-commerce Order Updated`: For complex order fulfillment or status updates.
    *   **Webhook URL:** This is the public HTTP POST endpoint URL of your deployed intermediary service. For testing, you can use services like `webhook.site` or `RequestBin` initially to inspect the Webflow payload.
    *   **Environment:** Choose "Live" for production site events or "Staging" for events on your staging domain.
5.  **Add Webhook:** Click "Add Webhook" to save the configuration.

**Example Webflow Form Submission Payload:**
When a `Form Submission` webhook is triggered, Webflow sends a JSON payload similar to this (exact fields depend on your form):

```json
{
  "triggerType": "form_submission",
  "data": {
    "_id": "60c7a8b4e1e3b6001d9c7c2b",
    "siteId": "60c7a8b4e1e3b6001d9c7c2a",
    "name": "Contact Us Form",
    "slug": "contact-us-form",
    "fieldData": {
      "Name": "Jane Doe",
      "Email": "jane.doe@example.com",
      "Subject": "Inquiry about services",
      "Message": "Please send me more information."
    },
    "submittedAt": "2023-10-27T10:30:00.000Z",
    "ipAddress": "192.0.2.1"
  }
}
```

## 3. Webhook Payload & Endpoint Configuration for Stripe

This section details how your intermediary service receives the Webflow webhook, processes its payload, and then interacts with the Stripe API.

**Intermediary Service Logic (Conceptual Code - Node.js Example):**

```javascript
// This represents a serverless function handler or an Express.js route
const express = require('express');
const Stripe = require('stripe');
const app = express();
app.use(express.json()); // Middleware to parse JSON request bodies

app.post('/webflow-stripe-webhook', async (req, res) => {
  const webflowPayload = req.body;
  const stripeSecretKey = process.env.STRIPE_SECRET_KEY; // Loaded securely from environment
  const stripe = new Stripe(stripeSecretKey);

  try {
    if (webflowPayload.triggerType === 'form_submission') {
      const { Name, Email, Message } = webflowPayload.data.fieldData;

      // Validate essential data for Stripe operations
      if (!Email) {
        return res.status(400).send('Email is required for Stripe operations.');
      }

      // 1. Logic: Find or Create a Customer in Stripe
      let customer;
      const existingCustomers = await stripe.customers.list({ email: Email, limit: 1 });
      if (existingCustomers.data.length > 0) {
        customer = existingCustomers.data[0];
        console.log(`Existing Stripe Customer found: ${customer.id}`);
      } else {
        customer = await stripe.customers.create({
          email: Email,
          name: Name || 'Unnamed Webflow User',
          description: `Customer from Webflow Form Submission (${webflowPayload.data.name})`,
          metadata: {
            webflow_form_id: webflowPayload.data._id,
            webflow_site_id: webflowPayload.data.siteId
          }
        });
        console.log(`New Stripe Customer created: ${customer.id}`);
      }

      // 2. Logic: Perform subsequent Stripe actions (e.g., create Payment Intent, subscription)
      // This is highly specific to your business needs and would involve more client-side interaction
      // for secure payment method collection (e.g., using Stripe.js).

    } else if (webflowPayload.triggerType === 'new_cms_item' && webflowPayload.data.collectionSlug === 'products') {
      // Example: Syncing a Webflow CMS 'Product' item to Stripe Products and Prices
      const { name, description, price, sku } = webflowPayload.data.fieldData;
      const stripeProduct = await stripe.products.create({
        name: name,
        description: description,
        active: true,
        metadata: {
          webflow_cms_item_id: webflowPayload.data._id,
          webflow_collection_slug: webflowPayload.data.collectionSlug
        }
      });
      await stripe.prices.create({
        unit_amount: price * 100, // Assuming 'price' is a dollar value from Webflow
        currency: 'usd',
        product: stripeProduct.id
      });
      console.log(`Stripe Product and Price created for CMS item: ${stripeProduct.id}`);
    } else {
      console.log(`Received unhandled Webflow trigger type: ${webflowPayload.triggerType}`);
      return res.status(202).send('No specific action configured for this trigger type.');
    }

    res.status(200).send('Webhook successfully processed.');

  } catch (error) {
    console.error('Error processing Webflow webhook:', error.message);
    res.status(500).send(`Error: ${error.message}`);
  }
});
// For serverless deployments, this app might be wrapped in a handler (e.g., module.exports.handler = app)
```

**Stripe API Request Body Examples (from your intermediary):**

*   **Create Customer:**
    ```json
    // POST https://api.stripe.com/v1/customers
    {
      "email": "jane.doe@example.com",
      "name": "Jane Doe",
      "description": "Customer from Webflow Form",
      "metadata": {
        "webflow_form_submission_id": "60c7a8b4e1e3b6001d9c7c2b"
      }
    }
    ```

*   **Create Product (from Webflow CMS item):**
    ```json
    // POST https://api.stripe.com/v1/products
    {
      "name": "My Webflow Product",
      "description": "Product synced from Webflow CMS.",
      "active": true,
      "metadata": {
        "webflow_cms_item_id": "60c7a8b4e1e3b6001d9c7c2d"
      }
    }
    ```

## 4. Testing and Validating the Data Sync

Thorough testing is paramount to ensure data integrity and reliable operation.

1.  **Intermediate Endpoint Deployment:** Deploy your intermediary serverless function or application. Ensure its public URL is accessible and configured correctly.
2.  **Initial Webflow Webhook Test:**
    *   **Tool:** Use a temporary webhook inspection service (e.g., `webhook.site`, `RequestBin`) as your Webflow webhook URL.
    *   **Action:** Trigger the configured event in Webflow (e.g., submit the relevant form, publish a CMS item).
    *   **Verification:** Confirm that the Webflow payload arrives correctly at the inspection service and matches the expected structure.
3.  **End-to-End Test with Stripe:**
    *   **Update Webflow Webhook URL:** Change the Webflow webhook URL to point to your *deployed* intermediary service endpoint.
    *   **Trigger Webflow Event:** Perform the action in Webflow that triggers the webhook.
    *   **Verification in Stripe Dashboard:**
        *   Log into your Stripe Dashboard (using test mode initially is highly recommended).
        *   Navigate to the relevant sections (e.g., Customers, Products, Payments) and verify that the data has been created or updated as expected.
        *   **Crucially, check the `Metadata` field** of the created Stripe objects. This should contain the Webflow IDs (e.g., `webflow_form_id`, `webflow_cms_item_id`) that your intermediary service included. This linking is vital for future data lookups or bi-directional synchronization.
    *   **Monitor Intermediary Logs:** Review the logs of your serverless function or application for successful API calls to Stripe and any errors encountered during processing. This provides real-time feedback on your integration's health.
4.  **Error Handling and Retries:**
    *   **Simulate Errors:** Temporarily introduce an error in your intermediary logic (e.g., pass an invalid Stripe API key, force a bad request) to observe how Webflow handles failed webhook deliveries. Webflow typically retries webhooks several times over 24 hours if it receives a non-2xx HTTP response from your endpoint.
    *   **Implement Robust Logging:** Ensure your intermediary service logs both successful operations and detailed error messages, including Webflow payload and Stripe API responses, to aid in debugging.
    *   **Idempotency:** Design your Stripe API calls to be idempotent where applicable (e.g., when creating customers, check if one with the same email already exists before creating a new one). This prevents duplicate entries if Webflow retries a webhook.
5.  **Security Review:**
    *   Ensure your Stripe Secret API Key is securely stored (environment variables) and never committed to version control.
    *   Consider implementing a shared secret for your Webflow webhooks if additional security is required beyond the obscurity of the URL. This would involve adding a query parameter to your Webflow webhook URL (e.g., `?auth_token=YOUR_SHARED_SECRET`) and validating it in your intermediary function.

By meticulously following these steps, you can establish a reliable and efficient data synchronization pipeline between Webflow and Stripe using raw webhooks and a custom intermediary service.