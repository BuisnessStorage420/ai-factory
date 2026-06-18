---
title: "Webflow to Supabase Webhook Setup Guide"
date: 2026-06-17
lastmod: 2026-06-17
publishdate: 2026-06-17
draft: false
---

# How to Sync Webflow with Supabase via Webhooks

This guide outlines the process for establishing a real-time data synchronization pipeline between Webflow CMS and Supabase using raw Webhooks. It provides a highly practical approach for ensuring data consistency across platforms.

## 1. Prerequisites (Authentication, Keys, API requirements)

Before initiating the synchronization process, ensure the following foundational elements are in place:

*   **Webflow Account:**
    *   An active Webflow site with CMS enabled.
    *   Administrative access to your Webflow project settings.
    *   Knowledge of the specific Collection(s) you intend to synchronize.
    *   *Optional but recommended:* A Webflow API token if your integration requires additional lookups beyond the webhook payload. While webhooks push data, a token allows for retrieving full item details or other related data if needed.

*   **Supabase Project:**
    *   An active Supabase project.
    *   Access to your Supabase project's `Project URL` and `API Keys` (specifically the `anon` public key for basic client-side access and the `service_role` key for server-side operations that bypass Row Level Security, used securely within Edge Functions).
    *   A target SQL table created in your Supabase project, with columns designed to accurately map to the Webflow CMS item fields you intend to store. For example, if syncing a "Blog Posts" collection, your Supabase table might have columns like `id` (primary key), `webflow_item_id`, `slug`, `name`, `published_on`, `_archived`, `_draft`, `data_json` (for flexible storage of the raw payload).
    *   Familiarity with Supabase Edge Functions (Deno-based serverless functions) for handling webhook requests securely and efficiently.

*   **General:**
    *   A fundamental understanding of HTTP requests, JSON data structures, and basic TypeScript/JavaScript.
    *   A secure method for storing and accessing environment variables (e.g., Supabase Secrets, .env files for local development).

## 2. Setting up the Trigger in Webflow

Webflow Webhooks allow you to send automated HTTP POST requests to a specified URL whenever certain events occur within your project.

1.  **Navigate to Webhooks Settings:**
    *   Log in to your Webflow project dashboard.
    *   Go to **Project Settings** > **Integrations**.
    *   Scroll down to the **Webhooks** section.

2.  **Add a New Webhook:**
    *   Click the **+ Add Webhook** button.

3.  **Configure the Webhook Details:**
    *   **Trigger Type:** Select the event that will initiate the webhook. Common triggers for data synchronization include:
        *   `Collection item created`
        *   `Collection item changed`
        *   `Collection item deleted`
        *   `Form submission` (if syncing form data)
        *   Ensure you select the specific CMS Collection you want to synchronize.
    *   **Webhook URL:** This is the endpoint where Webflow will send the POST request. For this integration, this will be the public URL of your Supabase Edge Function (e.g., `https://<YOUR_SUPABASE_PROJECT_REF>.supabase.co/functions/v1/webflow-webhook`).
    *   **API Version:** Select `v1`.
    *   **Secret:** Webflow generates a unique `Secret` for each webhook. **Copy this secret immediately.** It is crucial for verifying the authenticity of incoming requests in your Supabase Edge Function, preventing unauthorized payload processing. Store this secret securely in your Supabase project's environment variables.

4.  **Save the Webhook:**
    *   Click **Add Webhook**. Webflow will now trigger an HTTP POST request to your specified URL whenever the configured event occurs.

## 3. Webhook Payload & Endpoint Configuration for Supabase

This section details how to configure your Supabase environment to receive, validate, and process the Webflow webhook payloads. The primary method involves a Supabase Edge Function.

### Webflow Webhook Payload Structure

When a Webflow webhook is triggered, it sends a JSON payload containing information about the event and the associated data. A typical `Collection item created` or `changed` payload might look like this:

```json
{
  "triggerType": "collection_item_changed",
  "data": {
    "item": {
      "_id": "60a7e0e7a2b2c3d4e5f6a7b8",
      "name": "My Awesome Blog Post",
      "slug": "my-awesome-blog-post",
      "updated-on": "2023-10-27T10:00:00.000Z",
      "created-on": "2023-10-26T10:00:00.000Z",
      "_archived": false,
      "_draft": false,
      "fieldData": {
        "title": "My Awesome Blog Post",
        "post-body": "<p>This is the content of my blog post.</p>",
        "publish-date": "2023-10-27T00:00:00.000Z",
        "author": {
          "_id": "60a7e0e7a2b2c3d4e5f6a7b9",
          "name": "Jane Doe"
        }
      }
    },
    "collection": {
      "_id": "60a7e0e7a2b2c3d4e5f6a7c0",
      "name": "Blog Posts",
      "slug": "blog-posts"
    }
  },
  "site": {
    "_id": "60a7e0e7a2b2c3d4e5f6a7c1",
    "name": "My Webflow Site"
  }
}
```

### Supabase Edge Function for Webhook Handling

Create a new Edge Function (e.g., `webflow-webhook`) in your Supabase project. This function will serve as the webhook endpoint.

1.  **Create the Edge Function:**
    *   Use the Supabase CLI: `supabase functions new webflow-webhook`
    *   Deploy the function: `supabase functions deploy webflow-webhook`

2.  **Edge Function Code (`functions/webflow-webhook/index.ts`):**
    The function performs signature verification, parses the payload, and upserts data into your Supabase table.

    ```typescript
    import { serve } from 'https://deno.land/std@0.177.0/http/server.ts';
    import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';
    import { HmacSha256 } from 'https://deno.land/std@0.177.0/hash/sha256.ts';

    // Supabase client with service_role key to bypass RLS for direct inserts
    const supabase = createClient(
      Deno.env.get('SUPABASE_URL')!,
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
    );

    serve(async (req) => {
      if (req.method !== 'POST') {
        return new Response('Method Not Allowed', { status: 405 });
      }

      const webflowSecret = Deno.env.get('WEBFLOW_WEBHOOK_SECRET');
      if (!webflowSecret) {
        console.error('WEBFLOW_WEBHOOK_SECRET environment variable not set.');
        return new Response('Server Configuration Error', { status: 500 });
      }

      // Read raw body for signature verification
      const rawBody = await req.text();
      const signature = req.headers.get('x-webflow-signature');

      if (!signature) {
        console.warn('Missing X-Webflow-Signature header.');
        return new Response('Unauthorized - Missing Signature', { status: 401 });
      }

      // Verify the signature
      const hmac = new HmacSha256(webflowSecret);
      hmac.update(rawBody);
      const expectedSignature = hmac.toString();

      if (expectedSignature !== signature) {
        console.warn('Invalid X-Webflow-Signature.');
        return new Response('Unauthorized - Invalid Signature', { status: 401 });
      }

      // Parse the JSON payload
      const payload = JSON.parse(rawBody);
      const item = payload.data.item;
      const triggerType = payload.triggerType;

      if (!item) {
        console.warn('Webhook payload missing item data.');
        return new Response('Bad Request - Missing Item Data', { status: 400 });
      }

      try {
        let result;
        const tableName = 'webflow_blog_posts'; // Replace with your target table name

        if (triggerType === 'collection_item_deleted') {
          // Handle deletion: soft delete or actual delete
          result = await supabase
            .from(tableName)
            .update({ _archived: true, deleted_at: new Date().toISOString() }) // Soft delete example
            .eq('webflow_item_id', item._id);
            // .delete() // Hard delete example
            // .eq('webflow_item_id', item._id);
        } else {
          // Handle creation/update (upsert)
          result = await supabase
            .from(tableName)
            .upsert(
              {
                webflow_item_id: item._id,
                name: item.name,
                slug: item.slug,
                created_at: item['created-on'],
                updated_at: item['updated-on'],
                _archived: item._archived,
                _draft: item._draft,
                title: item.fieldData.title, // Map specific fields
                post_body: item.fieldData['post-body'],
                publish_date: item.fieldData['publish-date'],
                author_name: item.fieldData.author?.name, // Handle nested fields
                raw_data: payload // Store full raw payload for flexibility/auditing
              },
              { onConflict: 'webflow_item_id', ignoreDuplicates: false } // Upsert by webflow_item_id
            );
        }

        if (result.error) {
          console.error('Supabase operation failed:', result.error);
          return new Response(JSON.stringify({ error: result.error.message }), {
            status: 500,
            headers: { 'Content-Type': 'application/json' },
          });
        }

        console.log(`Successfully processed webhook for item ID: ${item._id}`);
        return new Response(JSON.stringify({ status: 'success' }), {
          status: 200,
          headers: { 'Content-Type': 'application/json' },
        });
      } catch (error) {
        console.error('Webhook processing error:', error);
        return new Response(JSON.stringify({ error: error.message }), {
          status: 500,
          headers: { 'Content-Type': 'application/json' },
        });
      }
    });
    ```

3.  **Environment Variables:**
    *   In your Supabase project dashboard, navigate to **Project Settings** > **Edge Functions** > **Secrets**.
    *   Add a new secret: `WEBFLOW_WEBHOOK_SECRET` with the value you copied from Webflow.
    *   Ensure your `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are correctly configured (they are usually set automatically for Edge Functions but confirm).

## 4. Testing and Validating the Data Sync

Thorough testing is critical to ensure data integrity and reliable synchronization.

1.  **Trigger a Test Event in Webflow:**
    *   Go to your Webflow CMS.
    *   For `collection_item_created` trigger: Create a new item in the designated collection.
    *   For `collection_item_changed` trigger: Edit an existing item and publish the changes.
    *   For `collection_item_deleted` trigger: Archive or delete an item.

2.  **Monitor Webflow Webhook Logs:**
    *   In Webflow, navigate back to **Project Settings** > **Integrations** > **Webhooks**.
    *   Click on the webhook you configured. You will see a list of recent deliveries, including their status (e.g., `200 OK`, `401 Unauthorized`, `500 Internal Server Error`). This provides an initial indication of whether the webhook fired successfully and if your Supabase endpoint responded.

3.  **Check Supabase Edge Function Logs:**
    *   In your Supabase project dashboard, navigate to **Edge Functions** > **Logs**.
    *   Select your `webflow-webhook` function. You should see logs indicating the successful receipt and processing of the webhook, including any `console.log` statements you added. Look for messages like "Successfully processed webhook for item ID..." or any error messages.

4.  **Verify Data in Supabase:**
    *   Open the **Table Editor** in your Supabase project.
    *   Browse the target table (e.g., `webflow_blog_posts`).
    *   Confirm that the new, updated, or deleted item's data is present and correctly mapped according to your Edge Function logic. Check specific fields, timestamps, and the `_archived` status for deletions.

5.  **Troubleshooting Common Issues:**
    *   **`401 Unauthorized` (Webflow logs):**
        *   **Signature Mismatch:** The `WEBFLOW_WEBHOOK_SECRET` in your Supabase Edge Function environment variables does not match the secret generated by Webflow. Double-check and update it.
        *   **Missing Signature Header:** Webflow might not be sending the header, or your function isn't reading it correctly.
    *   **`500 Internal Server Error` (Webflow logs/Supabase function logs):**
        *   **Supabase Client Errors:** Check if `SUPABASE_URL` or `SUPABASE_SERVICE_ROLE_KEY` are correct.
        *   **Table Name/Column Mismatch:** Ensure your `tableName` and column names in the `upsert` or `update` operation match your Supabase schema exactly.
        *   **Data Type Mismatches:** Ensure the data types pushed from Webflow align with your Supabase table column definitions (e.g., dates, booleans, text length).
        *   **RLS (Row Level Security):** If using the `anon` key instead of `service_role` in your Edge Function, ensure RLS policies permit the `INSERT`/`UPDATE` operations for the authenticated user, or disable RLS for that table if appropriate for the webhook context (generally not recommended). Using `service_role` securely within Edge Functions is the preferred method for bypassing RLS for server-side operations.
    *   **Data Not Appearing (Supabase):**
        *   Review Edge Function logs thoroughly for any unhandled exceptions or `console.error` messages.
        *   Confirm the `onConflict` clause in your `upsert` statement is correct (e.g., `webflow_item_id` as the conflict target).

By following these steps, you establish a robust, secure, and real-time data synchronization mechanism between Webflow and Supabase.