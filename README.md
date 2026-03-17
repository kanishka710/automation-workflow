# Shopify Automation Workflow for n8n

This repository contains a set of interconnected n8n workflows designed to automate customer engagement and sales recovery processes for e-commerce businesses using Shopify, Google services, Slack, and Twilio.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Workflows](#workflows)
   - [Post-Delivery Feedback Automation](#post-delivery-feedback-automation)
   - [Cart Abandonment Recovery](#cart-abandonment-recovery)
   - [Weekly Customer Feedback Summary](#weekly-customer-feedback-summary)
   - [Daily Cart/Checkout Recovery Summary](#daily-cartcheckout-recovery-summary)
3. [Setup and Configuration](#setup-and-configuration)
4. [Google Sheets Structures](#google-sheets-structures)
5. [Troubleshooting](#troubleshooting)
6. [Potential Enhancements](#potential-enhancements)

---

## Project Overview

This n8n automation suite aims to:

- Proactively gather customer feedback after order fulfillment.
- Automate recovery of abandoned shopping carts/checkouts.
- Provide automated summary reports to leadership and teams on customer satisfaction and recovery performance.

The system is modular, leveraging n8n's event-driven architecture and integrations for a robust customer engagement pipeline.

---

## Workflows

### 1. Post-Delivery Feedback Automation
<img width="1401" height="255" alt="image" src="https://github.com/user-attachments/assets/57e66b84-f2c2-4bb2-8eeb-f71da40c1101" />


**Goal:** Collect customer feedback after order fulfillment, analyze sentiment, and escalate critical issues.

- **Trigger:** Shopify `orders/fulfilled` webhook.
- **Flow:**  
  Shopify Trigger → Prepare Survey Link & Data → Wait (24h) → Send Survey Email (Gmail) → Google Sheets Trigger (on survey response) → Standardize Fields → Conditional Check (Rating) → Alert via Twilio (WhatsApp) or Slack

**Key Integrations:** Shopify, Google Forms, Google Sheets, Gmail, Twilio, Slack

---

### 2. Cart Abandonment Recovery
<img width="1253" height="274" alt="image" src="https://github.com/user-attachments/assets/5b1c07ba-15b8-460f-bd79-a71ab29c6ffa" />


**Goal:** Detect abandoned checkouts, send personalized recovery emails, and track recovery efforts.

- **Trigger:** Shopify `checkouts/create` webhook.
- **Flow:**  
  Shopify Trigger → Standardize Fields → Check Cart in Google Sheet → Update or Add Cart → Abandonment Checker (60 min inactivity) → Send Recovery Email (Gmail) → Log Outreach in Google Sheet

**Key Integrations:** Shopify, Google Sheets, Gmail

---

### 3. Weekly Customer Feedback Summary
<img width="552" height="261" alt="image" src="https://github.com/user-attachments/assets/b13f44f3-1a5f-4faf-8d84-24f0ce74c13a" />


**Goal:** Generate a weekly summary of customer feedback and send it to leadership via Slack and Gmail.

- **Trigger:** Scheduled Cron job (weekly).
- **Flow:**  
  Schedule Trigger → Get Feedback from Google Sheet → Calculate Stats → Post Summary to Slack → Email Digest to Leadership

**Key Integrations:** Google Sheets, Slack, Gmail

---

### 4. Daily Cart/Checkout Recovery Summary
<img width="624" height="232" alt="image" src="https://github.com/user-attachments/assets/43beb2dd-b451-436d-b1ae-f9a0820cbbb2" />


**Goal:** Generate a daily summary of cart/checkout recovery efforts and post it to Slack.

- **Trigger:** Scheduled Cron job (daily).
- **Flow:**  
  Schedule Trigger → Get Recovery Logs from Google Sheet → Calculate Stats → Post Summary to Slack

**Key Integrations:** Google Sheets, Slack

---

## Setup and Configuration

### Prerequisites

- n8n Instance (Self-hosted or n8n Cloud)
- Docker & Docker Compose (if self-hosting)
- ngrok (for local webhook exposure)
- Shopify Partner Account & Development Store
- Google Account (Forms, Sheets, Gmail)
- Twilio Account (for WhatsApp)
- Slack Workspace

### Environment Variables

Set the following for n8n (especially if self-hosting):

- `WEBHOOK_URL`, `N8N_PROTOCOL`, `N8N_HOST`, `N8N_PORT`, `VUE_APP_URL_BASE_API`, `N8N_EDITOR_BASE_URL`  
  (Use your ngrok URL for webhooks)

### Shopify Custom App

- Create a custom app in Shopify Admin.
- Grant necessary Admin API scopes:  
  - Read orders (for orders/fulfilled)
  - Read checkouts (for checkouts/create)
  - Read cart data (if used)
- Install app and copy Admin API access token for n8n.

### Google Forms & Apps Script

- Create a Google Form for feedback.
- Use "Get pre-filled link" to identify field IDs.
- In Google Sheets, use Apps Script to POST responses to n8n webhook.
- Set up onFormSubmit trigger in Apps Script.

### Google Sheets

- **Form Responses 1:** For post-delivery feedback (linked to Google Form).
- **Cart Status:** For abandoned cart tracking.

### Gmail, Twilio, Slack

- Set up OAuth2 credentials for Gmail in Google Cloud Console.
- Create Twilio account and WhatsApp sandbox for testing.
- Create Slack app, add bot token scopes, and invite to channels.

---

## Google Sheets Structures

### Form Responses 1 (Feedback)

| Column Header                                         | Data Type      | Purpose                                 |
|-------------------------------------------------------|----------------|-----------------------------------------|
| Timestamp                                             | Date/Time      | When the survey was submitted           |
| Overall, how satisfied are you with our product/service? | Number         | Customer's rating (1-5)                 |
| How likely are you to recommend us to a friend or colleague? | Number         | NPS-style question                      |
| Comments                                              | String         | Customer's open-ended feedback          |
| Order ID                                              | Number/String  | Pre-filled Shopify Order ID             |
| Customer Email                                        | String         | Pre-filled Customer Email               |
| Customer First Name                                   | String         | Pre-filled Customer First Name          |

### Cart Status (Abandoned Cart Tracking)

| Column Header         | Data Type      | Purpose                                  |
|----------------------|---------------|------------------------------------------|
| checkout_id/cart_id  | Number/String | Unique ID of the Shopify checkout/cart   |
| customer_email       | String        | Email of the customer                    |
| customer_phone       | Number/String | Phone number of the customer             |
| customer_first_name  | String        | First name of the customer               |
| created_at           | ISO 8601      | When the checkout was created            |
| last_updated_at      | ISO 8601      | Last activity timestamp                  |
| workflow_status      | String        | Status in the recovery workflow          |
| first_outreach_at    | ISO 8601      | Timestamp of first recovery message      |
| last_outreach_at     | ISO 8601      | Timestamp of most recent recovery message|
| outreach_attempts    | Number        | Count of recovery messages sent          |
| recovery_channel_sent| String        | Channel used for last outreach           |
| offer_code_sent      | String        | Discount code sent                       |
| recovered_order_id   | Number/String | Shopify Order ID if checkout recovered   |

---

## Troubleshooting

- **Shopify Trigger 422 Error:** Ensure correct API scopes in Shopify app.
- **Webhook Protocol Error:** Use HTTPS (ngrok or SSL).
- **ngrok Antivirus Flag:** Add ngrok.exe to antivirus exclusions.
- **Google Sheets/Apps Script:** Ensure correct field mapping and webhook URLs.
- **Twilio WhatsApp Format:** Use `whatsapp:+COUNTRYCODE_NUMBER` (no spaces).
- **Gmail/Google Sheets Auth:** Set up OAuth2 credentials and enable APIs.
- **Missing Data in Reports:** Ensure all relevant columns are updated by workflows.

---

## Potential Enhancements

- Multi-stage abandoned cart recovery (1h, 24h, 48h with escalating offers)
- AI-powered sentiment analysis (OpenAI, AWS Comprehend)
- Dynamic product recommendations from Shopify
- SMS/WhatsApp opt-in management
- Customizable report periods
- Error notifications to Slack
- Customer segmentation for tailored messaging

---

**For detailed node configurations and code snippets, see the workflow JSON files and in-line documentation.**
