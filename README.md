# Lithobee Automation Workflow

This repository contains a production-style n8n workflow export for Lithobee operations.
It automates abandoned-cart recovery, post-delivery feedback collection, low-rating alerts, and scheduled reporting.

## What This Automation Covers

1. Abandoned cart capture and recovery outreach
2. Post-delivery feedback request email
3. Low satisfaction alerting to Slack (and optional WhatsApp via Twilio)
4. Daily cart recovery KPI reporting to Slack
5. Weekly feedback summary to Slack and Gmail
6. Recovery reconciliation when a previously abandoned cart becomes paid

## Workflow File

- Main workflow export: [Lithobee Automation.json](Lithobee%20Automation.json)

## Visual Block Views

These are the workflow block screenshots available in this folder.

### Block View 1

![Workflow Block View 1](Screenshot%202026-03-17%20222426.png)

### Block View 2

![Workflow Block View 2](Screenshot%202026-03-17%20222635.png)

### Block View 3

![Workflow Block View 3](Screenshot%202026-03-17%20222701.png)

### Block View 4

![Workflow Block View 4](Screenshot%202026-03-17%20223906.png)

### Block View 5

![Workflow Block View 5](Screenshot%202026-03-17%20223921.png)

## High-Level Architecture

### Trigger Layer

- Shopify Trigger- checkout created
- Shopify Trigger-order fullfilled
- Shopify Trigger - Order Paid
- Google Sheets Trigger - Feedback Row Added
- Schedule - Daily Report
- Schedule - Weekly Report

### Processing Layer

- Set nodes normalize and shape payloads for downstream systems
- Code nodes parse feedback payloads and compute summary metrics
- IF - Low Rating branches alert and non-alert paths
- Wait nodes create timed delay windows before outreach

### Persistence Layer

- Google Sheets - Append Cart writes new abandoned cart rows
- Google Sheets - Update Cart updates outreach metadata
- Google Sheets - Mark Cart Recovered reconciles paid orders
- Google Sheets read nodes source report data

### Notification Layer

- Gmail - Cart Recovery
- Gmail - Feedback Form Link
- Slack - Low Rating Alert
- Slack - Daily Report
- Slack - Weekly Report
- Gmail - Weekly Digest
- Send an SMS/MMS/WhatsApp message (Twilio, currently disabled)

## Detailed Flow Breakdown

## 1) Abandoned Cart Recovery Flow

### Purpose

Capture abandoned checkouts, persist them, send recovery email after delay, then update tracking fields.

### Sequence

1. Shopify Trigger- checkout created fires
2. Set - Cart Row maps checkout data into a structured row payload
3. Google Sheets - Append Cart inserts row into Abandoned Carts sheet
4. Wait - 60 Min Cart delays first outreach
5. Gmail - Cart Recovery sends reminder email with recovery link
6. Code - Cart Update Payload sets status and outreach timestamps
7. Google Sheets - Update Cart writes outreach state back by checkout_id

### Key Data Fields

- checkout_id
- customer_email
- customer_phone
- customer_first_name
- recovery_url
- workflow_status
- outreach_attempts
- recovery_channel_sent
- offer_code_sent

## 2) Post-Delivery Feedback + Alert Flow

### Purpose

Request customer feedback after fulfillment and instantly alert if satisfaction is low.

### Sequence A: Feedback request after fulfillment

1. Shopify Trigger-order fullfilled fires on fulfillment
2. Wait - 24h Post Delivery delays request timing
3. Set - Feedback Form Link builds personalized Google Form URL
4. Gmail - Feedback Form Link sends feedback request email

### Sequence B: Feedback ingestion and low-rating branch

1. Google Sheets Trigger - Feedback Row Added detects new response row
2. Code - Parse Form Payload normalizes rating/comment fields
3. IF - Low Rating evaluates satisfaction_rating < 3
4. True path:
   - Slack - Low Rating Alert posts alert
   - Send an SMS/MMS/WhatsApp message can send WhatsApp alert when enabled
5. False path:
   - No-op - Positive Feedback ends without alert

### Alert Criteria

- Low rating threshold: satisfaction_rating less than 3

## 3) Daily Cart Recovery Report Flow

### Purpose

Produce daily operational visibility into cart recovery outcomes.

### Sequence

1. Schedule - Daily Report runs at cron 0 8 * * *
2. Google Sheets - Read Cart (Daily) fetches cart tracking rows
3. Code - Daily Summary filters entries for current IST day and computes:
   - total checkouts
   - outreach sent count
   - recovered count
   - pending count
   - recovery rate percentage
4. Slack - Daily Report posts summary to daily-reports channel

## 4) Weekly Feedback Report Flow

### Purpose

Create weekly customer feedback digest for operations and leadership.

### Sequence

1. Schedule - Weekly Report runs at cron 0 9 * * 1
2. Config provides admin_email used by digest email node
3. Google Sheets - Read Feedback (Weekly) reads feedback rows
4. Code - Weekly Summary filters last 7 days and calculates:
   - total responses
   - average satisfaction
   - average NPS
   - low-rating count
5. Fan-out outputs to:
   - Slack - Weekly Report
   - Gmail - Weekly Digest

## 5) Cart Recovery Confirmation Flow

### Purpose

Reconcile abandoned cart records once payment is completed.

### Sequence

1. Shopify Trigger - Order Paid fires
2. Code - Mark Cart Recovered maps checkout_id and recovered status
3. Google Sheets - Mark Cart Recovered updates matching row by checkout_id

## External Systems and Dependencies

1. n8n instance (workflow runtime)
2. Shopify store with OAuth app credentials
3. Google account access to both sheets
4. Gmail OAuth connection for outbound emails
5. Slack OAuth connection and target channels
6. Optional Twilio credentials for WhatsApp alerting

## Google Sheets Used

1. Feedback responses sheet
   - Document ID: 1K-1hPyJKUXCGlqfPBKymAz6CCmM-2Ev0iud8_VYPL2c
   - Sheet: Form responses 1
2. Abandoned cart tracking sheet
   - Document ID: 1Wg2rQJ6rLUaAJGwa9YZJ59ysKrkEPUitb0_QKuebwL8
   - Sheet: Abandoned Carts

## Slack Channels Referenced

1. daily-reports
2. weekly-reports
3. low-rating-alerts

## Configuration and Environment

## Node-Level Config

- Config node currently sets admin_email to kanishkachaturvedi07@gmail.com

## Environment Variables

Twilio node references:

- TWILIO_WHATSAPP_FROM (default fallback in node)
- TWILIO_WHATSAPP_TO_ALERTS

## Current Operational State

1. Workflow active flag is false in export
2. Twilio alert node is disabled

## How To Import and Run

1. Open n8n
2. Import [Lithobee Automation.json](Lithobee%20Automation.json)
3. Reconnect all credentials in imported nodes
4. Verify both Google Sheets are accessible and column names match expected schema
5. Validate Slack channel bindings
6. Optionally enable and configure Twilio node
7. Set workflow to active
8. Run each branch with test payloads before production activation

## Test Checklist

1. Create a checkout in Shopify and verify row append + recovery email path
2. Mark an order fulfilled and verify delayed feedback email path
3. Add mock low rating in feedback sheet and verify Slack low-rating alert
4. Trigger daily schedule manually and verify daily report format
5. Trigger weekly schedule manually and verify Slack + Gmail digest
6. Trigger order paid event and verify workflow_status updates to recovered

## Troubleshooting Guide

## If cart rows are not updating

1. Confirm checkout_id values are present and consistent across append and update paths
2. Ensure Google Sheets update matching column is checkout_id
3. Confirm OAuth token for Google Sheets is valid

## If feedback alert is not firing

1. Verify parsed satisfaction_rating is numeric
2. Confirm IF - Low Rating condition is still less than 3
3. Check whether responses are entering through the expected sheet and trigger

## If scheduled reports do not send

1. Confirm workflow is active
2. Verify schedule node timezone assumptions against your operations timezone
3. Validate Slack OAuth and channel permissions

## Security and Operations Notes

1. Keep OAuth credentials managed only in n8n credential vault
2. Avoid storing secrets in plain-text nodes when possible
3. Restrict sheet and channel access to required scopes only
4. Add retry/error workflows for production hardening if required

## Recommended Enhancements

1. Add dead-letter/error handling branch per critical integration node
2. Add idempotency checks for duplicate checkout or feedback events
3. Add alert suppression logic to prevent duplicate low-rating notifications
4. Add dashboarding layer using BI tool for historical trend visualization

## Ownership

This README documents the current exported workflow behavior and expected operations for deployment and maintenance.
