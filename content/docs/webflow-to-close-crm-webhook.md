---
title: "Webflow to Close CRM Webhook Setup Guide"
date: 2026-06-17
lastmod: 2026-06-17
publishdate: 2026-06-17
draft: false
---

# How to Sync Webflow with Close CRM via Webhooks

This guide details the process of synchronizing data between Webflow and Close CRM using raw Webhooks, requiring an intermediary custom listener application. This method offers granular control over data transformation and flow.

## 1. Prerequisites (Authentication, Keys, API requirements)

Before commencing, ensure the following foundational elements are in place:

*   **Webflow Account:**
    *   A paid Webflow plan (Basic, CMS, Business, Enterprise) is required to enable Webhooks functionality.
    *   Access to the Webflow Project Settings for the site you intend to integrate.
    *   Identification of the specific Webflow Form or CMS Collection that will act as the data source.
*   **Close CRM Account:**
    *   Active Close CRM subscription with administrator privileges.
    *   **API Key Generation:** Navigate to `Settings` > `API Keys` in Close CRM. Generate a new API Key. This key is crucial for authenticating requests to the Close CRM API. Keep it secure.
    *   **API Key Format:** Close CRM API utilizes Basic Authentication. Your API Key must be Base64 encoded, typically in the format `base64(YOUR_API_KEY:)`.
*   **Custom Webhook Listener/Server:**
    *   A dedicated server, cloud function (e.g., AWS Lambda, Google Cloud Functions, Azure Functions), or custom application capable of:
        *   Receiving HTTP POST requests from Webflow.
        *   Parsing JSON payloads.
        *   Making authenticated HTTP POST/PUT requests to the Close CRM API.
        *   Logging requests and responses for debugging.
    *   This listener will serve as the "bridge" between Webflow's outbound webhook and Close CRM's inbound API. You will need a publicly accessible URL for this listener.
*   **Understanding of Webflow & Close CRM API Documentation:**
    *   Familiarity with Webflow's Webhook payload structures (e.g., Form Submission, CMS Item Created).
    *   Understanding of Close CRM's Lead, Contact, and Custom Field API endpoints and their required JSON request bodies.

## 2. Setting up the Trigger in Webflow

Configure Webflow to send a webhook payload to your custom listener whenever a specific event occurs.

1.  **Navigate to Project Settings:** In your Webflow project, go to `Project Settings` > `Integrations`.
2.  **Access Webhooks:** Scroll down to the `Webhooks` section.
3.  **Add New Webhook:** Click `Add Webhook`.
4.  **Configure Webhook Details:**
    *   **Name:** Provide a descriptive name (e.g., "New Lead Form Submission to Close").
    *   **Trigger Type:** Select the event that will fire the webhook. Common triggers include:
        *   `Form Submission`: For capturing data from a specific form.
        *   `CMS Item Created`: For new entries in a CMS Collection.
        *   `CMS Item Updated`: For modifications to existing CMS entries.
        *   `CMS Item Deleted`: For removing CMS entries.
    *   **Filter (if applicable):** If `Form Submission` is selected, specify the exact Form Name you want to monitor. For CMS events, select the relevant Collection.
    *   **Webhook URL:** Enter the full, publicly accessible URL of your custom webhook listener application (e.g., `https://your-custom-listener.com/webflow-close-webhook`).
5.  **Add Webhook:** Click `Add Webhook` to save your configuration.
6.  **Test Webhook (Optional but Recommended):** Webflow provides a `Send Test` button next to your configured webhook. This sends a dummy payload to your specified URL, allowing you to verify your listener is reachable and correctly processing the initial request. Use a tool like `webhook.site` temporarily if your listener isn't fully developed, to inspect the test payload structure.

**Example Webflow Form Submission Payload (partial):**

```json
{
  "_id": "60c72f10d2d3a4b5c6d7e8f0",
  "triggerType": "form_submission",
  "siteId": "60c72f10d2d3a4b5c6d7e8f1",
  "data": {
    "name": "New Lead Submission",
    "slug": "contact-form",
    "formId": "60c72f10d2d3a4b5c6d7e8f2",
    "formData": {
      "name": "John Doe",
      "email": "john.doe@example.com",
      "phone": "+1234567890",
      "company": "Acme Corp",
      "message": "Interested in product X."
    },
    "submittedAt": "2023-10-27T10:00:00.000Z",
    "ipAddress": "192.168.1.1"
  }
}
```

## 3. Webhook Payload & Endpoint Configuration for Close CRM

Your custom listener application will intercept the Webflow webhook, transform its payload, and then interact with the Close CRM API.

1.  **Receive and Parse Webflow Payload:**
    *   Your listener receives the HTTP POST request from Webflow.
    *   Extract and parse the JSON body.
    *   Identify the relevant data points from the `data.formData` (for forms) or `data.current` (for CMS items) object.
2.  **Map Webflow Data to Close CRM Fields:**
    *   Determine which Webflow fields correspond to Close CRM fields (e.g., Webflow `name` to Close CRM `contact.name`, Webflow `email` to `contact.emails`).
    *   Consider creating or updating a Lead, Contact, or Activity.
    *   For new Leads, Close CRM requires at least one Contact.
3.  **Construct Close CRM API Request Body:**
    *   Based on your mapping, build the JSON payload required by the Close CRM API.
    *   **Endpoint Example: Create Lead with Contact**
        *   **Method:** `POST`
        *   **URL:** `https://api.close.com/api/v1/lead/`
        *   **Headers:**
            *   `Content-Type: application/json`
            *   `Authorization: Basic {Base64_Encoded_API_Key}` (e.g., `Basic YXBrXzXXXXXXXXXXXXXXXpOg==`)
        *   **Body Example:**
            ```json
            {
              "name": "Acme Corp Lead", // Often derived from Webflow 'company' or 'name'
              "description": "Lead generated via Webflow form submission.",
              "status": "status_your_status_id", // Replace with actual status ID from Close CRM
              "contacts": [
                {
                  "name": "John Doe", // From Webflow formData.name
                  "emails": [
                    {
                      "email": "john.doe@example.com", // From Webflow formData.email
                      "type": "work"
                    }
                  ],
                  "phones": [
                    {
                      "phone": "+1234567890", // From Webflow formData.phone
                      "type": "mobile"
                    }
                  ]
                }
              ]
              // Add custom fields if necessary:
              // "custom.custom_field_id": "Webflow_Value"
            }
            ```
            *   *Note:* Close CRM `status` and `custom fields` require their specific IDs, obtainable via the Close CRM API (e.g., `/api/v1/status/` or `/api/v1/custom_field/`).
4.  **Make API Call to Close CRM:**
    *   Using your custom listener, send the constructed HTTP POST request to the Close CRM API endpoint.
    *   Handle potential network errors, API rate limits, and non-2xx HTTP responses from Close CRM. Log errors meticulously.

## 4. Testing and Validating the Data Sync

Thorough testing is critical to ensure data integrity and reliable synchronization.

1.  **Configure Logging:**
    *   Ensure your custom webhook listener has robust logging for both incoming Webflow payloads and outgoing Close CRM API requests/responses. This is invaluable for debugging.
    *   For initial testing, tools like `webhook.site` or `Postman` can simulate the Webflow webhook or test Close CRM API calls directly.
2.  **Trigger Webflow Event:**
    *   **For Form Submissions:** Go to your live Webflow site and submit the form configured with the webhook. Use realistic test data.
    *   **For CMS Events:** In the Webflow Editor, create a new CMS item, update an existing one, or publish changes to trigger the respective webhook.
3.  **Monitor Your Listener:**
    *   Observe the logs of your custom listener. Verify that it received the Webflow webhook payload successfully.
    *   Check for correct parsing and transformation of the data.
    *   Confirm that the outgoing Close CRM API request body was correctly constructed and sent.
    *   Look for the HTTP status code from the Close CRM API response (a `200 OK` or `201 Created` indicates success).
4.  **Validate in Close CRM:**
    *   Log in to your Close CRM account.
    *   Search for the lead/contact/activity you expected to be created or updated based on your test data.
    *   Verify that all mapped fields (name, email, phone, company, custom fields) have been populated accurately.
    *   Confirm that the lead/contact is associated with the correct status or other organizational elements.
5.  **Troubleshooting Common Issues:**
    *   **Webhook Delivery Failure:** Check Webflow `Project Settings` > `Integrations` > `Webhooks` for delivery logs and any errors reported by Webflow. Ensure your listener URL is correct and publicly accessible.
    *   **Payload Transformation Errors:** Review your listener's logs to ensure the Webflow payload is correctly parsed and that your field mapping logic is robust.
    *   **Close CRM API Authentication:** Double-check the Base64 encoding of your Close CRM API Key and ensure it's correctly included in the `Authorization` header.
    *   **Close CRM API Response Errors:** Analyze the error messages from Close CRM (e.g., `400 Bad Request`, `401 Unauthorized`, `404 Not Found`). These often indicate issues with the request body structure, missing required fields, or incorrect IDs for statuses/custom fields.
    *   **Rate Limiting:** If performing many operations, be aware of Close CRM's API rate limits. Implement exponential backoff or queuing in your listener for production systems.
6.  **Implement Idempotency and Error Handling (Production Consideration):**
    *   For production environments, design your listener to handle duplicate Webflow webhook deliveries (idempotency) to prevent creating duplicate leads.
    *   Implement robust error handling, including retries for transient Close CRM API errors and notifications for critical failures.