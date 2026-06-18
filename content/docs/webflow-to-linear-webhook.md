---
title: "Webflow to Linear Webhook Setup Guide"
date: 2026-06-17
lastmod: 2026-06-17
publishdate: 2026-06-17
draft: false
---

# How to Sync Webflow with Linear via Webhooks

As a Principal Integration Engineer, this guide outlines the process of synchronizing data between Webflow CMS and Linear issues using raw webhooks, requiring an intermediary processing layer. This approach ensures robust, custom-tailored data flow and transformation.

## 1. Prerequisites (Authentication, Keys, API requirements)

Before initiating the synchronization, ensure the following foundational components and access permissions are in place:

**Webflow:**
*   **Active Webflow Site:** A live Webflow site with a CMS collection (e.g., "Feature Requests," "Bug Reports") from which data will be sent.
*   **Webflow Hosting Plan:** A Webflow hosting plan that supports CMS and Webhooks (typically CMS or Business hosting).
*   **Admin Access:** Administrator access to the Webflow project to configure webhooks.
*   **Collection Structure:** An understanding of the specific fields within your Webflow CMS collection that need to be mapped to Linear.

**Linear:**
*   **Linear Workspace:** An active Linear workspace.
*   **Personal API Key:** Generate a Personal API Key from your Linear settings (`Settings > API > Personal API Keys`). This key is crucial for authenticating requests to the Linear GraphQL API. Store this key securely; it grants broad access to your Linear data.
*   **Team ID & Project ID:** Identify the specific Linear Team ID and, optionally, a Project ID where issues will be created. These can be found by inspecting the URL in Linear (e.g., `linear.app/<team-id>/project/<project-id>/view`) or by querying the Linear API.

**Intermediary Service (Webhook Listener & Processor):**
Webflow webhooks send data, but Linear's API requires a specific GraphQL mutation. A direct call is not possible; therefore, an intermediary service is mandatory. This service will:
*   Receive the incoming Webflow webhook payload.
*   Parse and transform the Webflow data into a Linear-compatible format.
*   Authenticate and send a GraphQL mutation to the Linear API.
*   **Publicly Accessible Endpoint:** This service must expose a publicly accessible HTTPS endpoint (e.g., a custom backend, AWS Lambda function, Google Cloud Function, or a platform like Vercel Functions).

**Development Environment:**
*   A suitable development environment for building your intermediary service (e.g., Node.js, Python, Go, Ruby). Knowledge of HTTP requests, JSON parsing, and GraphQL is essential.

## 2. Setting up the Trigger in Webflow

Configure Webflow to send a webhook payload whenever a specified event occurs within your CMS.

1.  **Navigate to Project Settings:** In your Webflow project, go to `Project Settings`.
2.  **Access Integrations:** Click on the `Integrations` tab.
3.  **Webhooks Section:** Scroll down to the `Webhooks` section and click `Add Webhook`.
4.  **Configure Webhook Details:**
    *   **Name:** Provide a descriptive name (e.g., "New Feature Request to Linear").
    *   **Trigger Type:** Select the event that should initiate the webhook. For creating new Linear issues, `Collection item created` is ideal. For updates, use `Collection item updated`.
    *   **Collection:** Choose the specific Webflow CMS collection you wish to monitor (e.g., "Feature Requests").
    *   **Webhook URL:** This is the `HTTPS` endpoint URL of your intermediary service. This URL must be publicly accessible and capable of receiving `POST` requests.
    *   **HTTP Method:** Set this to `POST`.
5.  **Add Webhook:** Click `Add Webhook` to save the configuration.

After setup, any new item created in the selected Webflow CMS collection (and published) will trigger a POST request to your specified Webhook URL with a JSON payload containing the item's data.

## 3. Webhook Payload & Endpoint Configuration for Linear

This section details the structure of the Webflow webhook payload and how to configure your intermediary service to process it and interact with the Linear API.

**Webflow Webhook Payload Structure (Example: `Collection item created`)**

When a webhook is triggered, Webflow sends a JSON payload similar to this. The `item.fieldData` object contains your specific CMS fields.

```json
{
  "triggerType": "collection_item_created",
  "webhookId": "65b2a0c4f8d5f30e0a5d4d3d",
  "data": {
    "site": {
      "_id": "65b2a0c4f8d5f30e0a5d4d3c",
      "name": "My Webflow Project"
    },
    "collection": {
      "_id": "65b2a0c4f8d5f30e0a5d4d3b",
      "name": "Feature Requests",
      "slug": "feature-requests"
    },
    "item": {
      "_id": "65b2a0c4f8d5f30e0a5d4d3a",
      "name": "Implement Dark Mode",
      "slug": "implement-dark-mode",
      "updatedOn": "2024-01-25T12:00:00.000Z",
      "createdOn": "2024-01-25T12:00:00.000Z",
      "publishedOn": "2024-01-25T12:00:00.000Z",
      "fieldData": {
        "name": "Implement Dark Mode",
        "short-description": "Users request an interface with a dark theme.",
        "priority": "High",
        "status": "New Request"
      }
    }
  }
}
```

**Intermediary Service Logic:**

Your intermediary service will perform the following steps:

1.  **Receive Webhook:** Listen for `POST` requests on your configured Webhook URL.
2.  **Parse Payload:** Parse the incoming JSON body. Access relevant data from `data.item.fieldData` (e.g., `data.item.fieldData.name`, `data.item.fieldData.short-description`).
3.  **Map Data:** Map Webflow fields to Linear issue fields.
    *   Webflow `fieldData.name` -> Linear `title`
    *   Webflow `fieldData.short-description` -> Linear `description`
    *   Hardcode or dynamically fetch `teamId` and `projectId`.
4.  **Construct Linear GraphQL Mutation:** Create a GraphQL mutation for `issueCreate`.

    ```javascript
    // Example pseudo-code for your intermediary service
    const webflowItemName = payload.data.item.fieldData.name;
    const webflowItemDescription = payload.data.item.fieldData['short-description']; // Use bracket notation for hyphens
    const linearTeamId = "YOUR_LINEAR_TEAM_ID"; // Get this from Linear settings/API
    const linearProjectId = "YOUR_LINEAR_PROJECT_ID"; // Optional, get from Linear

    const linearMutation = {
      query: `
        mutation IssueCreate($title: String!, $description: String, $teamId: String!, $projectId: String) {
          issueCreate(
            input: {
              title: $title,
              description: $description,
              teamId: $teamId,
              projectId: $projectId
            }
          ) {
            success
            issue {
              id
              title
              url
            }
            lastSyncId
          }
        }
      `,
      variables: {
        title: webflowItemName,
        description: webflowItemDescription,
        teamId: linearTeamId,
        projectId: linearProjectId
      }
    };
    ```

5.  **Send Request to Linear API:**
    *   **Endpoint:** `https://api.linear.app/graphql`
    *   **HTTP Method:** `POST`
    *   **Headers:**
        *   `Authorization: Bearer YOUR_LINEAR_API_KEY` (replace with your actual API key)
        *   `Content-Type: application/json`
    *   **Body:** The `linearMutation` JSON object constructed above.

**Example Linear API Request Body:**

```json
{
  "query": "mutation IssueCreate($title: String!, $description: String, $teamId: String!, $projectId: String) { issueCreate(input: { title: $title, description: $description, teamId: $teamId, projectId: $projectId }) { success issue { id title url } lastSyncId } }",
  "variables": {
    "title": "Implement Dark Mode",
    "description": "Users request an interface with a dark theme.",
    "teamId": "f0170092-23c3-4d2c-91d4-1234567890ab",
    "projectId": "8b9f0103-67c4-4e3a-9e0c-abcdef123456"
  }
}
```

## 4. Testing and Validating the Data Sync

Thorough testing is critical to ensure the integration functions as expected.

1.  **Trigger the Webhook in Webflow:**
    *   Navigate to your Webflow Designer.
    *   Go to your CMS collection (e.g., "Feature Requests").
    *   Create a new item with sample data, ensuring all mapped fields are populated.
    *   **Publish your Webflow site.** Webhooks are only sent after publishing.

2.  **Monitor the Intermediary Service:**
    *   Check the logs of your intermediary service. Look for:
        *   Confirmation of receiving the Webflow webhook.
        *   Successful parsing and data mapping.
        *   The constructed Linear GraphQL mutation.
        *   The HTTP request and response status code from the Linear API.
        *   Any error messages.

3.  **Verify in Linear:**
    *   Log into your Linear workspace.
    *   Navigate to the team and/or project where the issue should have been created.
    *   Confirm that a new issue with the title and description from your Webflow item has appeared.
    *   Verify that all mapped data fields are correct.

4.  **Troubleshooting Common Issues:**
    *   **Webhook Not Firing:** Ensure the Webflow site was published after creating the CMS item. Check Webflow Project Settings > Integrations > Webhooks for delivery attempts and statuses (red usually indicates failure).
    *   **Intermediary Service Not Receiving:** Verify your Webhook URL is correct, publicly accessible, and configured to accept POST requests on the specified port/path. Check server firewall rules.
    *   **Payload Parsing Errors:** Review your intermediary service code to ensure it correctly parses the Webflow JSON structure. Pay attention to case sensitivity and nested objects.
    *   **Linear API Errors:**
        *   Check your intermediary service logs for specific error messages from Linear (e.g., "Invalid API Key," "Team not found," "Required field missing").
        *   Ensure your Linear Personal API Key is correct and has the necessary permissions.
        *   Confirm the `teamId` and `projectId` in your GraphQL mutation are accurate.
        *   Validate the GraphQL query syntax using Linear's API Explorer if necessary.
    *   **Data Mismatch:** Review your data mapping logic in the intermediary service. Ensure the correct Webflow `fieldData` keys are being used and mapped to the appropriate Linear fields.

By meticulously following these steps and addressing any encountered issues, you can establish a robust, real-time synchronization between your Webflow CMS and Linear via raw webhooks, empowering efficient workflow automation.