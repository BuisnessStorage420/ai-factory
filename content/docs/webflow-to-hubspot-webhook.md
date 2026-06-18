---
title: "Webflow to HubSpot Webhook Setup Guide"
date: 2026-06-17
lastmod: 2026-06-17
publishdate: 2026-06-17
draft: false
---

# How to Sync Webflow with HubSpot via Webhooks

This guide details the technical process of synchronizing data between Webflow and HubSpot using raw Webhooks, focusing on precision and practical implementation. This method typically involves an intermediary service to receive Webflow payloads, transform them, and then make authenticated requests to the HubSpot API.

## 1. Prerequisites (Authentication, Keys, API requirements)

Successful integration requires specific access and understanding of both platforms' API mechanisms.

**Webflow:**
*   **Project Access:** Administrative access to your Webflow project is necessary to configure Webhooks for Forms or CMS Collections.
*   **No Direct Authentication:** Webflow's native Webhooks do not require or offer direct authentication for arbitrary target URLs. They simply POST data to the configured endpoint.

**HubSpot:**
*   **HubSpot Account:** An active HubSpot account with permissions to create/update contacts and access API keys.
*   **Private App Access Token:** This is the recommended and most secure authentication method for HubSpot's CRM API.
    1.  Navigate to `Settings > Integrations > Private Apps`.
    2.  Create a new private app, providing a name and description.
    3.  Define the necessary API scopes. For contact creation/updates, typically `crm.objects.contacts.write` and `crm.objects.contacts.read` are required.
    4.  Generate and securely store the "Access token" for your private app. This token will be used in the `Authorization` header (`Bearer <ACCESS_TOKEN>`) for all requests to the HubSpot CRM API.
*   **CRM API Endpoint:** The primary endpoint for creating or updating contacts is `https://api.hubapi.com/crm/v3/objects/contacts`.
*   **Property Knowledge:** Familiarity with HubSpot contact properties (e.g., `firstname`, `lastname`, `email`, `phone`, `company`) is essential for mapping data correctly.

**Intermediary Service:**
Given Webflow's webhook limitations and HubSpot's API authentication requirements, an intermediary service is critical. This service acts as a bridge, receiving the unauthenticated Webflow webhook, processing its payload, and then making an authenticated call to HubSpot.
*   **Platform:** Serverless functions (AWS Lambda, Google Cloud Functions, Azure Functions) or a custom backend application are ideal for this.
*   **Webhook Endpoint:** This service must expose a public HTTP POST endpoint to receive the Webflow webhook payload.
*   **Development Environment:** Tools for development (e.g., Node.js, Python runtime), a text editor, and version control.

## 2. Setting up the Trigger in Webflow

Configure Webflow to dispatch a webhook when a specific event occurs.

**A. Form Submission Webhooks:**
This is a common trigger for syncing lead data.
1.  In your Webflow project, navigate to **Project Settings > Integrations**.
2.  Scroll down to the **Webhooks** section.
3.  Click **Add Webhook**.
4.  **Trigger Type:** Select `Form Submission`.
5.  **Webhook URL:** Enter the public HTTP POST endpoint URL of your intermediary service.
6.  Click **Add Webhook**.

**B. CMS Item Webhooks:**
For syncing content or status updates.
1.  In your Webflow project, navigate to **CMS** and select the desired Collection (e.g., Blog Posts, Products).
2.  Go to the **Settings** tab for that Collection.
3.  Scroll down to the **Webhooks** section.
4.  Click **Add Webhook**.
5.  **Trigger Type:** Select the desired event (e.g., `Item Created`, `Item Updated`, `Item Published`).
6.  **Webhook URL:** Enter the public HTTP POST endpoint URL of your intermediary service.
7.  Click **Add Webhook**.

**Example Webflow Form Submission Payload (JSON):**
When a form is submitted, Webflow sends a POST request with a JSON body similar to this:

```json
{
  "name": "My Contact Form",
  "siteId": "651a1b2c3d4e5f6a7b8c9d0e",
  "data": {
    "name": "John Doe",
    "email": "john.doe@example.com",
    "phone": "123-456-7890",
    "message": "Interested in your services.",
    "_f_name": "My Contact Form",
    "_website": "My Website Name",
    "_source": "https://www.example.com/contact"
  },
  "orderedData": [
    {
      "name": "name",
      "value": "John Doe"
    },
    {
      "name": "email",
      "value": "john.doe@example.com"
    },
    {
      "name": "phone",
      "value": "123-456-7890"
    },
    {
      "name": "message",
      "value": "Interested in your services."
    }
  ],
  "d": "2023-10-26T10:00:00.000Z",
  "id": "653a2b1c4d5e6f7a8b9c0d1e"
}
```
The `data` object contains the form field names and their submitted values.

## 3. Webhook Payload & Endpoint Configuration for HubSpot

The intermediary service is responsible for translating the Webflow payload into a HubSpot API-compatible request.

**Intermediary Service Logic:**
1.  **Receive Webhook:** The service's endpoint receives the HTTP POST request from Webflow.
2.  **Parse Payload:** Extract the JSON body from the incoming request.
3.  **Data Extraction & Mapping:** Identify and extract relevant fields from the Webflow payload (`data` object for forms, `fieldData` for CMS items). Map these to corresponding HubSpot contact properties.
    *   For instance, `webflow_payload.data.name` might be split into `firstname` and `lastname` for HubSpot.
    *   `webflow_payload.data.email` maps directly to HubSpot's `email`.
4.  **Construct HubSpot API Request Body:** Create a JSON object adhering to the HubSpot CRM API `create/update contact` schema. The `email` property is crucial for HubSpot to identify existing contacts and prevent duplicates (it acts as a unique identifier for creation, and can be used for updates with certain endpoints or by first searching for the contact).

**Example HubSpot Contact Creation Payload (JSON):**

```json
{
  "properties": {
    "email": "john.doe@example.com",
    "firstname": "John",
    "lastname": "Doe",
    "phone": "123-456-7890",
    "hs_lead_status": "New Lead",
    "webflow_form_name": "My Contact Form"
  }
}
```
*Note: Custom properties like `webflow_form_name` must be pre-created in HubSpot.*

5.  **Send Authenticated Request to HubSpot:**
    *   **Method:** `POST`
    *   **URL:** `https://api.hubapi.com/crm/v3/objects/contacts`
    *   **Headers:**
        ```
        Authorization: Bearer YOUR_HUBSPOT_PRIVATE_APP_ACCESS_TOKEN
        Content-Type: application/json
        ```
    *   **Body:** The constructed JSON payload (as shown above).
6.  **Error Handling & Logging:** Implement robust logging to capture the success or failure of the HubSpot API call, along with any error messages. This is critical for debugging and monitoring.

## 4. Testing and Validating the Data Sync

Thorough testing ensures data flows correctly and reliably.

**A. Initial Webhook Payload Inspection:**
1.  **Use a Webhook Inspector:** Before pointing your Webflow Webhook to your intermediary service, configure it to send to a service like `webhook.site` or `requestbin.com`.
2.  **Trigger Event:** Submit a form or publish a CMS item in Webflow.
3.  **Inspect Payload:** Examine the payload received by the inspector tool. Verify that all expected data fields are present and correctly formatted. This step confirms Webflow is sending the data as anticipated.

**B. Intermediary Service Testing:**
1.  **Deploy Service:** Deploy your intermediary service to its public endpoint.
2.  **Direct Test (Optional but Recommended):** Use a tool like Postman, Insomnia, or cURL to send a sample Webflow-like JSON payload directly to your intermediary service's endpoint. This allows for isolated testing of your data transformation and HubSpot API interaction logic.
    ```bash
    curl -X POST \
      https://your-intermediary-service.com/webhook-handler \
      -H 'Content-Type: application/json' \
      -d '{
            "name": "Test Form",
            "data": {
              "name": "Jane Test",
              "email": "jane.test@example.com",
              "phone": "098-765-4321"
            }
          }'
    ```
3.  **Monitor Logs:** Observe the logs of your intermediary service to confirm it received the request, processed the data, and successfully made the call to HubSpot. Look for any errors returned by the HubSpot API.

**C. End-to-End Validation:**
1.  **Update Webflow Webhook:** Change the Webhook URL in Webflow to point to your deployed intermediary service.
2.  **Trigger Live Event:** Perform the actual trigger event in Webflow (e.g., submit the live form on your website).
3.  **Verify in HubSpot:**
    *   Navigate to your HubSpot Contacts.
    *   Search for the contact created/updated by your test submission (e.g., using the email address).
    *   Verify that all mapped properties (first name, last name, email, phone, custom fields) are populated correctly with the data from Webflow.
4.  **Edge Case Testing:** Test with different data variations:
    *   Missing optional fields.
    *   Data with special characters.
    *   Submitting a form with an email that already exists in HubSpot (to verify update logic or duplicate handling).

**D. Ongoing Monitoring:**
Implement monitoring for your intermediary service to detect failures in real-time. This includes monitoring service health, API call success rates, and specific error logs to ensure continuous data synchronization.