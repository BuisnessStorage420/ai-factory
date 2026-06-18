---
title: "Webflow to Mailchimp Webhook Setup Guide"
date: 2026-06-17
lastmod: 2026-06-17
publishdate: 2026-06-17
draft: false
---

# How to Sync Webflow with Mailchimp via Webhooks

This guide details the process of synchronizing data, specifically form submissions, from Webflow to Mailchimp utilizing raw webhooks. This method requires an intermediary service to transform the Webflow payload into a format consumable by Mailchimp's API.

## 1. Prerequisites (Authentication, Keys, API requirements)

To establish a robust data synchronization pipeline, ensure the following components and credentials are in place:

*   **Webflow Account:**
    *   Access to a Webflow project with an active site and at least one form designed for data collection (e.g., newsletter signup, contact form).
    *   Publishing permissions for the Webflow site to enable webhook triggers.

*   **Mailchimp Account:**
    *   An active Mailchimp account with an Audience (formerly "List") designated for new subscribers.
    *   **Mailchimp API Key:** Obtainable from your Mailchimp account under `Profile > Extras > API Keys`. This key is essential for authenticating API requests.
    *   **Mailchimp Audience ID:** Locate this by navigating to `Audience > Settings > Audience name and default settings` within your Mailchimp dashboard. The ID will be listed under "Audience ID".
    *   **Mailchimp API Endpoint:** The base URL for Mailchimp's API is typically `https://<dc>.api.mailchimp.com/3.0/`, where `<dc>` is your data center (e.g., `us1`, `eu2`).

*   **Webhook Processing Service (Intermediary):**
    *   Mailchimp does not directly consume arbitrary JSON payloads from webhooks to add subscribers. Instead, it requires structured API calls. Therefore, an intermediary service is mandatory. This service will act as the target for Webflow's webhook, parse its payload, transform the data, and then make an authenticated API request to Mailchimp.
    *   Examples of such services include:
        *   **No-code/Low-code platforms:** Zapier, Make.com (formerly Integromat), Pipedream.
        *   **Serverless Functions:** AWS Lambda, Azure Functions, Google Cloud Functions (requiring custom code).
        *   **Custom Backend Service:** A dedicated server endpoint (e.g., Node.js, Python, PHP) hosted on a cloud provider.
    *   For this guide, we assume a generic "webhook endpoint" provided by one of these services, capable of receiving a `POST` request and executing custom logic.

## 2. Setting up the Trigger in Webflow

Configure Webflow to dispatch a webhook whenever a specific event occurs, such as a form submission.

1.  **Navigate to Project Settings:** In your Webflow project, go to `Project Settings`.
2.  **Access Integrations:** Click on the `Integrations` tab.
3.  **Locate Webhooks:** Scroll down to the `Webhooks` section.
4.  **Add New Webhook:** Click the `Add Webhook` button.
5.  **Configure Webhook Details:**
    *   **Name:** Provide a descriptive name (e.g., "Mailchimp Sync - Newsletter Form").
    *   **Trigger Type:** Select `Form Submission`. This ensures the webhook fires every time any form on your site is submitted. (Note: For specific forms, the intermediary service will filter by `formId` or `formName` within the payload).
    *   **Webhook URL:** This is the URL of your intermediary service's endpoint, which is designed to receive Webflow's webhook payload. This URL must be publicly accessible.

    ```
    Example Webhook URL: https://your-intermediary-service.com/webflow-mailchimp-handler
    ```
6.  **Add Webhook:** Click `Add Webhook` to save the configuration.
7.  **Publish Your Site:** Ensure your Webflow site is published for the webhook to become active.

Webflow will now send an HTTP `POST` request with a JSON payload to the specified Webhook URL upon every form submission.

## 3. Webhook Payload & Endpoint Configuration for Mailchimp

This section outlines what Webflow sends and what Mailchimp expects, detailing the role of the intermediary service.

### Understanding Webflow's Form Submission Payload

Upon a form submission, Webflow sends a `POST` request with a JSON body similar to this:

```json
{
  "triggerId": "653b6a9a8f2f4e001b1a2b3c",
  "triggerType": "form_submission",
  "dataType": "formData",
  "data": {
    "_id": "653b6ab50a0a0a0a0a0a0a0a",
    "siteId": "653b6a9a8f2f4e001b1a2b3c",
    "formId": "653b6a9a8f2f4e001b1a2b3d",
    "formName": "Newsletter Signup",
    "submittedAt": "2023-10-27T14:30:00.000Z",
    "ipAddress": "192.168.1.1",
    "fieldData": {
      "Name": "Jane Doe",
      "Email": "jane.doe@example.com",
      "Consent": "true"
    }
  }
}
```

Key fields for Mailchimp synchronization:
*   `data.formName` or `data.formId`: Useful for routing if you have multiple forms.
*   `data.fieldData.Email`: The subscriber's email address.
*   `data.fieldData.Name`: Often contains the full name, which may need parsing into first and last names for Mailchimp's `FNAME` and `LNAME` merge fields.

### Mailchimp API Endpoint & Request Structure

To add or update a subscriber in Mailchimp, you'll make an authenticated `POST` request to the `members` endpoint.

*   **Endpoint URL:** `https://<dc>.api.mailchimp.com/3.0/lists/{audience_id}/members`
    *   Replace `<dc>` with your Mailchimp data center (e.g., `us1`).
    *   Replace `{audience_id}` with your Mailchimp Audience ID.

*   **Authentication:** Mailchimp uses HTTP Basic Authentication.
    *   **Username:** Any string (conventionally, "anystring" or "apikey").
    *   **Password:** Your Mailchimp API Key.

*   **Request Body (JSON):** The request body must conform to Mailchimp's subscriber schema.

```json
{
  "email_address": "jane.doe@example.com",
  "status": "subscribed",
  "merge_fields": {
    "FNAME": "Jane",
    "LNAME": "Doe"
  },
  "tags": [
    "Webflow Form",
    "Newsletter Signup"
  ],
  "interests": {
    "your_group_id": true
  }
}
```
*   `email_address`: Required. The email from `data.fieldData.Email`.
*   `status`: Recommended. Can be `subscribed`, `pending` (for double opt-in), `unsubscribed`, or `cleaned`. `subscribed` is common for direct sync.
*   `merge_fields`: Optional, but highly recommended for personalization. Map fields like `Name` from Webflow to `FNAME` and `LNAME` in Mailchimp. You may need to parse the `Name` field.
*   `tags`: Optional. An array of strings to categorize subscribers.
*   `interests`: Optional. For grouping subscribers based on their preferences.

### Intermediary Service Logic (Conceptual)

Your intermediary service will perform the following steps:

1.  **Receive Webflow Webhook:** Accept the `POST` request from Webflow.
2.  **Parse Payload:** Extract the JSON body from Webflow's request.
3.  **Extract Data:**
    *   `webflowEmail = payload.data.fieldData.Email`
    *   `webflowName = payload.data.fieldData.Name` (e.g., "Jane Doe")
    *   *Transformation:* Split `webflowName` into `firstName` ("Jane") and `lastName` ("Doe").
4.  **Construct Mailchimp Request Body:** Create a JSON object adhering to Mailchimp's subscriber requirements using the extracted and transformed data.
5.  **Make Mailchimp API Call:** Send an authenticated `POST` request to the Mailchimp `members` endpoint.
    *   Include the Basic Auth header: `Authorization: Basic <base64_encoded("anystring:YOUR_API_KEY")>`
    *   Set `Content-Type: application/json`.
6.  **Handle Responses:** Process Mailchimp's API response. Check for success (HTTP 200/201) or specific error codes (e.g., 400 for existing member, 401 for authentication issues). Update logs accordingly.

## 4. Testing and Validating the Data Sync

Thorough testing ensures data flows correctly and reliably.

1.  **Webflow Form Submission:**
    *   Publish your Webflow site to ensure the webhook is active.
    *   Navigate to your live Webflow site.
    *   Fill out the form (e.g., newsletter signup) with test data (a unique email address).
    *   Submit the form.

2.  **Webflow Webhook Logs:**
    *   In your Webflow Project Settings, go to `Integrations > Webhooks`.
    *   Check the "Trigger Logs" section for your configured webhook.
    *   Verify that the webhook fired successfully (status 200 or 2xx). If there's an error, it indicates an issue with Webflow sending the webhook to your intermediary.

3.  **Intermediary Service Logs (Crucial):**
    *   Access the logs of your webhook processing service.
    *   Confirm that your service received the webhook payload from Webflow.
    *   Verify that the service correctly parsed the Webflow data and constructed the Mailchimp API request.
    *   Check for any errors during the API call to Mailchimp. This is where most integration issues occur (e.g., incorrect API key, malformed request body, Mailchimp validation errors).
    *   *Debugging Tip:* Temporarily point your Webflow webhook to a debugging tool like `webhook.site` or `requestbin.com` to inspect the exact payload Webflow sends. This helps isolate issues between Webflow's sending and your intermediary's receiving.

4.  **Mailchimp Audience Validation:**
    *   Log into your Mailchimp account.
    *   Navigate to the specific Audience that your webhook should be populating.
    *   Search for the test email address you used in step 1.
    *   Verify that the subscriber has been added, and check that `merge_fields` (like First Name, Last Name) and any assigned `tags` are correctly populated.
    *   If the subscriber appears but with incorrect data, revisit your intermediary service's data transformation logic. If the subscriber does not appear, review the Mailchimp API response in your intermediary's logs for error messages.

By following these steps, you can establish a robust and accurate data synchronization between Webflow forms and Mailchimp audiences using raw webhooks and an essential intermediary service.