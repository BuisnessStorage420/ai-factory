---
title: "Webflow to Xero"
date: 2026-06-17
---

# Webflow to Xero Webhooks: A Step-by-Step Integration Guide

Automating workflows between your website and accounting software can save countless hours and reduce manual errors. This guide will walk you through setting up a webhook integration to connect Webflow form submissions directly to Xero, allowing you to automatically create contacts, generate invoices, or update records based on user interactions on your site.

## Introduction to Webflow-Xero Webhooks

Webhooks are automated messages sent from an application when a specific event occurs. In this scenario, when a user submits a form on your Webflow site, Webflow will send a webhook containing that form data. However, Xero's API expects data in a very specific format. This means we'll need an intermediary service to "listen" for the Webflow webhook, transform the data into a structure Xero understands, and then send it to the Xero API. This powerful integration can streamline processes like lead generation, customer onboarding, and sales invoicing.

## Prerequisites

Before diving into the setup, ensure you have the following:

1.  **Webflow Account & Site:** A live Webflow site with at least one form designed to collect the data you wish to send to Xero (e.g., customer name, email, service details, amount).
2.  **Xero Account:** An active Xero account with appropriate permissions to create contacts or invoices. Familiarity with your Xero chart of accounts is helpful, especially for invoicing.
3.  **An Intermediary Automation Service:** This is crucial. Webflow sends raw form data, but Xero expects structured API calls (JSON/XML). You'll need a tool to bridge this gap. Popular choices include:
    *   **Make.com (formerly Integromat):** Highly visual and flexible.
    *   **Zapier:** User-friendly, good for simpler integrations.
    *   **Pipedream / n8n:** More developer-centric, offering greater control.
    *   *(For this guide, we'll use a generic "intermediary service" approach, as the core concepts are similar across platforms.)*
4.  **Basic Understanding of JSON:** While not strictly necessary for simple setups with tools like Zapier, a basic grasp of JSON data structures will help with debugging and complex mappings.
5.  **Xero API Access:** Ensure your intermediary service can authenticate with your Xero account, usually via OAuth 2.0.

## Understanding the Workflow

The typical flow for this integration looks like this:

1.  **Webflow Form Submission:** A user fills out and submits a form on your Webflow site.
2.  **Webflow Webhook Trigger:** Webflow fires a webhook containing the form data.
3.  **Intermediary Service Listener:** Your chosen automation service (e.g., Make.com) receives this webhook.
4.  **Data Transformation:** The intermediary service extracts the relevant data, maps it to Xero's required fields, and formats it appropriately.
5.  **Xero API Call:** The intermediary service then makes an API call to Xero (e.g., creating a new contact or an invoice).
6.  **Xero Action:** Xero processes the API call and performs the requested action.

---

## 1. Setting Up the Webflow Trigger

The first step is to tell Webflow when and where to send its data.

1.  **Navigate to Webflow Project Settings:**
    *   Open your Webflow project dashboard.
    *   Click the "Project Settings" icon (gear icon) for the site you're working on.
2.  **Go to Integrations Tab:**
    *   In the Project Settings, click on the "Integrations" tab.
3.  **Add a New Webhook:**
    *   Scroll down to the "Webhooks" section.
    *   Click "Add Webhook".
4.  **Configure Webhook Details:**
    *   **Event Type:** Select "Form Submission". If you have multiple forms, you can specify a particular form from the dropdown. This ensures the webhook only fires for the intended form.
    *   **Webhook URL:** This is the critical piece. Go to your intermediary service (e.g., Make.com, Zapier) and set up a "Webhook Listener" or "Catch Hook" module. This module will generate a unique URL. Copy that URL and paste it here.
        *   *Example in Make.com:* Add a "Webhooks" module, choose "Custom Webhook," and click "Create a webhook." Make.com will provide the URL.
        *   *Example in Zapier:* Set "Webhooks by Zapier" as your trigger, select "Catch Hook," and copy the URL.
    *   **Method:** Keep this as "POST" (default for form submissions).
    *   **Headers (Optional):** You typically won't need custom headers for a basic Webflow form submission webhook.
5.  **Add Webhook:** Click the "Add Webhook" button to save your configuration.

### Crucial Step: Initial Webhook Test

With your webhook configured in Webflow and your intermediary service's listener awaiting data, you need to send a *sample submission*.

1.  Go to your live Webflow site where the form is published.
2.  Fill out the form with sample data (e.g., "John Doe," "john.doe@example.com," "Test Product," "100").
3.  Submit the form.

This action will send data to your intermediary service, allowing it to "learn" the data structure. This is essential for mapping fields in the next step. Verify in your intermediary service that the data was received.

---

## 2. Configuring the Intermediary Service (Payload Transformation)

This is where you bridge the gap between Webflow's raw data and Xero's structured API requirements. Your intermediary service will receive the Webflow data and then process it before sending it to Xero.

1.  **Receive Webhook Data:**
    *   Your webhook listener module (from the previous step) should now show that it has successfully received data from your Webflow form submission. Inspect this data to understand its structure (e.g., `data.name`, `data.email`, `data.amount`).

2.  **Add Xero Action Module:**
    *   Connect a new module to your webhook listener that interacts with Xero. Depending on your goal, this could be:
        *   **Xero - Create a Contact:** To add new customers or leads.
        *   **Xero - Create an Invoice:** To generate sales invoices.
        *   **Xero - Search for a Contact:** Often needed *before* creating an invoice, to link it to an existing customer or decide if a new one needs to be created.
    *   **Authenticate Xero:** You'll be prompted to connect your Xero account. Follow the OAuth flow to grant your intermediary service access.

3.  **Map Webflow Data to Xero Fields:**
    This is the core of the transformation. You'll drag-and-drop or type in references to the data received from Webflow into the corresponding Xero fields.

    ### Example 1: Creating a Xero Contact from a Webflow Form
    *   **Xero Module:** "Create a Contact"
    *   **Fields Mapping:**
        *   `Name`: Map from your Webflow form field (e.g., `{{webhook.data.name}}` or `{{1.data.name}}` depending on your service).
        *   `EmailAddress`: Map from Webflow (e.g., `{{webhook.data.email}}`).
        *   `FirstName`, `LastName`: You might need to parse `name` if your form only collects a full name, or map directly if you have separate first/last name fields.
        *   `BankAccountDetails`, `TaxNumber`, etc.: Map if your form collects these.

    ### Example 2: Creating a Xero Sales Invoice from a Webflow Form
    This is more complex as it often requires a contact first.

    *   **Step A: Search/Create Contact (Pre-Invoice)**
        *   **Module:** "Xero - Search Contacts"
        *   **Search by:** `EmailAddress` (map from Webflow `{{webhook.data.email}}`).
        *   **Module:** Conditional logic or "Router" to check if a contact was found. If not found, add "Xero - Create a Contact" module (mapping fields as above). This ensures invoices are linked to the correct customer.
        *   Store the `ContactID` from either the search result or the newly created contact.

    *   **Step B: Create Invoice**
        *   **Module:** "Xero - Create an Invoice"
        *   **Type:** `ACCREC` (for a sales invoice).
        *   **Contact:** Select "Custom" or map the `ContactID` obtained from Step A.
        *   **Date:** Can be dynamic (e.g., `now` or `parseDate("now")`).
        *   **DueDate:** Dynamic (e.g., `addDays(now, 30)` for 30 days).
        *   **LineItems (Array):** This is where you detail the products/services. You'll typically click "Add item" and map:
            *   `Description`: Map from Webflow (e.g., `{{webhook.data.service_description}}`).
            *   `Quantity`: Map from Webflow (e.g., `{{webhook.data.quantity}}`) or set to `1`.
            *   `UnitAmount`: Map from Webflow (e.g., `{{webhook.data.price}}`).
            *   `AccountCode`: **Crucial for Xero.** This must be a valid revenue account code from your Xero Chart of Accounts (e.g., `200` for "Sales").
            *   `TaxType`: (e.g., `OUTPUT` for standard sales tax, `NONE` if no tax applies).
        *   **CurrencyCode:** (e.g., `USD`, `AUD`, `NZD`) if needed.

    ### Key Considerations for Payload Transformation:
    *   **Required Fields:** Xero API has mandatory fields. Ensure all are mapped correctly.
    *   **Data Types:** Make sure the data type matches (e.g., amount is a number, date is a date format). Some intermediary services handle this automatically.
    *   **Error Handling:** Consider what happens if a field is empty, or if an invalid value is sent. Many services offer error handling features.
    *   **Conditional Logic:** For complex scenarios (e.g., different invoice types based on form selection), use routers or conditional paths in your intermediary service.

---

## 3. Testing the End-to-End Workflow

Thorough testing is vital to ensure your integration works reliably.

1.  **Enable Your Scenario/Flow:** Make sure your intermediary service's scenario (e.g., "Zap," "Scenario," "Workflow") is turned "ON" and actively listening.
2.  **Submit a Test Form:**
    *   Go to your live Webflow site.
    *   Fill out the form with *realistic test data*. Use names/emails you can easily identify in Xero (e.g., "TEST COMPANY - John Doe").
    *   Submit the form.
3.  **Verify in Intermediary Service:**
    *   Immediately check the execution history or logs of your intermediary service.
    *   Look for a successful run (usually indicated by green checkmarks or a "success" status).
    *   If there's an error, the logs will often provide details on what went wrong (e.g., "Invalid AccountCode," "Contact not found").
4.  **Verify in Xero:**
    *   Log in to your Xero account.
    *   Navigate to "Contacts" or "Business" > "Invoices" (depending on your setup).
    *   Search for the contact or invoice you just created using your test data.
    *   **Crucially, check all mapped fields:**
        *   Is the contact name correct?
        *   Is the invoice amount accurate?
        *   Are the line items, account code, and tax type correctly applied?
        *   Is the invoice in the correct status (e.g., Draft, Awaiting Approval)?
5.  **Iterate and Refine:** If you encounter errors or incorrect data, go back to your intermediary service, adjust the mappings or logic, and re-test. Repeat until perfect.

### Troubleshooting Tips:

*   **Webhook URL:** Double-check that the Webflow webhook URL matches the one from your intermediary service.
*   **Data Mapping:** Ensure the correct Webflow fields are mapped to the correct Xero API fields, paying attention to case sensitivity and data types.
*   **Xero API Requirements:** Refer to the official Xero API documentation for specific requirements of the endpoints you're using (e.g., mandatory fields for invoices).
*   **Authentication:** Confirm your Xero connection in the intermediary service is still valid. Sometimes tokens expire.
*   **Test Data:** Always use test data that mimics real-world scenarios but won't impact your actual accounting records.

---

## Conclusion

By leveraging Webflow webhooks and an intermediary automation service, you can create a robust and efficient bridge between your website and Xero accounting software. This integration not only eliminates manual data entry and reduces human error but also empowers you to build highly responsive and automated business processes directly from your Webflow site. Start simple, test thoroughly, and gradually expand your automation capabilities to unlock the full potential of your digital ecosystem.