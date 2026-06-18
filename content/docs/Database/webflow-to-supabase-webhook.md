---
title: "Webflow to Supabase"
date: 2026-06-17
---

This guide will walk you through setting up a webhook in Webflow to send form submission data directly to your Supabase database. This powerful integration allows you to automate data collection, build custom backend logic, and manage your Webflow data more flexibly.

---

# Webflow to Supabase Webhooks: A Comprehensive Guide

Integrating Webflow forms with Supabase via webhooks opens up a world of possibilities for custom data handling, backend logic, and scalable application development. This guide will provide a step-by-step walkthrough to connect your Webflow form submissions directly to a Supabase table.

## Introduction

Webhooks are automated messages sent from one application to another when a specific event occurs. In this scenario, when a user submits a form on your Webflow site, a webhook will fire, sending the form data to a designated API endpoint in your Supabase project. Supabase, acting as a backend-as-a-service, can then store this data directly into a table, trigger database functions, or interact with other services.

## Prerequisites

Before diving in, ensure you have the following ready:

1.  **A Webflow Account and Project:** A published Webflow site with at least one form designed.
2.  **A Supabase Account and Project:** A Supabase project set up and running.
3.  **Basic Understanding of JSON:** Webhooks typically send data in JSON format.
4.  **Basic API Concepts:** Familiarity with POST requests, headers, and API endpoints.

## 1. Prepare Your Supabase Database

First, we need to set up your Supabase project to receive data.

### 1.1 Create Your Supabase Table

You'll need a table in Supabase to store the form submissions. Ensure the column names in your Supabase table exactly match the `name` attributes of your form fields in Webflow.

**Example Table Schema (e.g., `contacts` table):**

```sql
CREATE TABLE public.contacts (
    id uuid DEFAULT gen_random_uuid() NOT NULL,
    name text,
    email text,
    message text,
    created_at timestamp with time zone DEFAULT now()
);

ALTER TABLE public.contacts ENABLE ROW LEVEL SECURITY;
```

### 1.2 Configure Row Level Security (RLS)

By default, Supabase tables have RLS enabled, which prevents unauthorized access. For public form submissions, you'll need to create a policy that allows anonymous users to `INSERT` data.

1.  Navigate to your Supabase project.
2.  Go to **Authentication > Policies**.
3.  Select your table (e.g., `contacts`).
4.  Click **New policy**.
5.  Choose **"Create a policy from scratch"**.
6.  **Name:** `Allow anonymous insert` (or similar).
7.  **Forced:** Check this.
8.  **Target Roles:** Select `anon`.
9.  **Permissive:** Check this.
10. **USING Expression:** Leave empty (`true`).
11. **WITH CHECK Expression:** Leave empty (`true`).
12. **Allowed Operations:** Check `INSERT`.
13. Click **Review** and then **Save policy**.

**Important:** Disabling RLS entirely or allowing `anon` users to `SELECT` or `UPDATE` could expose your data. Only enable `INSERT` for the `anon` role for public-facing forms.

### 1.3 Get Your Supabase API Credentials

You'll need your project's URL and anonymous public key to authenticate your webhook request.

1.  In your Supabase project, go to **Project Settings > API**.
2.  Locate your **Project URL** (e.g., `https://[YOUR_PROJECT_REF].supabase.co`).
3.  Locate your **`anon` (public) key** under "Project API keys".

## 2. Create Your Webflow Form

Design your form in Webflow as you normally would. The critical step here is to ensure your input fields have the correct **Name** attribute. These names *must* match the column names in your Supabase table.

**Example Webflow Form Setup:**

*   **Text Input Field for Name:**
    *   **Name:** `name`
*   **Email Input Field for Email:**
    *   **Name:** `email`
*   **Textarea for Message:**
    *   **Name:** `message`
*   **Submit Button**

## 3. Configure the Webflow Webhook Trigger

Now, let's tell Webflow to send data to Supabase when the form is submitted.

1.  In your Webflow Designer, go to **Project Settings** (the cog icon).
2.  Navigate to the **Integrations** tab.
3.  Scroll down to the **Webhooks** section.
4.  Click **"Add new webhook"**.

    *   **Name:** Give your webhook a descriptive name (e.g., "Supabase Contact Form Submission").
    *   **Trigger Type:** Select **"Form Submission"**. This corresponds to the `WEBFLOW_FORM_SUBMIT_WEBHOOK` event.
    *   **Webhook URL:** This is the API endpoint for your Supabase table.
        *   Use the format: `https://[YOUR_PROJECT_REF].supabase.co/rest/v1/[YOUR_TABLE]`
        *   **Example:** `https://abcdefg12345.supabase.co/rest/v1/contacts`
        *   Replace `[YOUR_PROJECT_REF]` with your actual Supabase Project Reference and `[YOUR_TABLE]` with your table name (e.g., `contacts`).
    *   **HTTP Method:** Select `POST`.

### 3.1 Configure Webhook Headers

You need to add specific headers for Supabase to authenticate the request and correctly interpret the data.

Under the "Headers" section for your new webhook, click "Add Header" for each of the following:

1.  **Header 1:**
    *   **Key:** `Authorization`
    *   **Value:** `Bearer [YOUR_ANON_KEY]`
        *   Replace `[YOUR_ANON_KEY]` with your `anon` (public) key obtained from Supabase.
2.  **Header 2:**
    *   **Key:** `apikey`
    *   **Value:** `[YOUR_ANON_KEY]`
        *   Again, replace `[YOUR_ANON_KEY]` with your `anon` (public) key.
3.  **Header 3:**
    *   **Key:** `Content-Type`
    *   **Value:** `application/json`

After adding all headers, click **"Add Webhook"** at the bottom.

## 4. Webhook Payload Configuration

Webflow automatically structures the payload for form submissions. When you select "Form Submission" as the trigger type, Webflow gathers all the input fields from the submitted form and converts them into a JSON object.

**How Webflow Generates the Payload:**

If your Webflow form has fields with `name` attributes like `name`, `email`, and `message`, the JSON payload sent by Webflow will look similar to this:

```json
{
  "name": "John Doe",
  "email": "john.doe@example.com",
  "message": "This is a test message from my Webflow form.",
  "formName": "Contact Form 1", // Webflow includes form metadata
  "date": "2023-10-27T10:30:00.000Z",
  "siteId": "...",
  "pageId": "...",
  // ... other Webflow-specific metadata
}
```

Supabase's REST API is designed to directly insert data when the JSON keys match the table's column names. Because we ensured our Webflow form field `name` attributes match our Supabase table column names (e.g., `name`, `email`, `message`), Supabase will automatically pick out those matching fields from the incoming JSON and insert them into the corresponding columns.

You don't need to manually configure the payload structure within Webflow for a direct form submission webhook; Webflow handles this for you.

## 5. Testing the Webhook

It's time to see your integration in action!

1.  **Publish your Webflow site.** Webhooks only trigger on the published site.
2.  Navigate to the page containing your form on your *published site*.
3.  Fill out the form with some test data.
4.  Click the **Submit** button.

### 5.1 Verify in Supabase

1.  Go to your Supabase project dashboard.
2.  Navigate to the **Table Editor** (the spreadsheet icon).
3.  Select your table (e.g., `contacts`).
4.  You should see a new row populated with the data you submitted from your Webflow form!

### 5.2 Troubleshooting

If you don't see the data:

*   **Check Webflow Webhook Logs:**
    *   In Webflow Project Settings > Integrations > Webhooks, click on your webhook.
    *   You'll see a log of recent webhook attempts, including their status (Success/Failure) and the response from the receiving server (Supabase in this case). Look for errors here.
*   **Check Supabase Logs:**
    *   In your Supabase project, go to **Logs Explorer** (the chart icon).
    *   Filter for `HTTP` requests to your table to see if the request reached Supabase and any errors it might have encountered (e.g., RLS violations, schema mismatches).
*   **Review RLS Policy:** Double-check that your `INSERT` policy for the `anon` role on your table is correctly configured. This is a very common cause of failures.
*   **Match Field Names:** Ensure the `name` attribute of each input field in your Webflow form *exactly* matches the column name in your Supabase table (case-sensitive).
*   **API Keys & URL:** Verify that your Supabase Project URL, Anon Key, and the full Webhook URL are correct and free of typos.
*   **Content-Type Header:** Ensure the `Content-Type: application/json` header is correctly set.

## Conclusion

Congratulations! You've successfully configured a Webflow webhook to send form submission data directly to your Supabase database. This setup forms the foundation for countless applications, from simple contact forms to complex data collection systems and user registration flows. With this integration, you can leverage Webflow's powerful design capabilities with Supabase's robust backend features, opening up exciting possibilities for your web projects.