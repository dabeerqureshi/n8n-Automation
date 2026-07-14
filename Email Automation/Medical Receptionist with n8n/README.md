# 🏥 AI Medical Receptionist Email Automation (n8n)

An AI-powered email automation workflow built with **n8n** that acts as a virtual medical receptionist. It automatically reads incoming patient emails, understands their intent using AI (OpenAI or Ollama), checks external systems like Google Calendar, generates professional replies, creates Gmail drafts, and logs patient interactions.

The workflow helps clinics reduce manual work while ensuring every patient receives a fast and consistent response.

---

## ✨ Features

- 📧 Monitor incoming Gmail messages automatically
- 🧹 Clean HTML emails and extract readable text
- 🤖 AI-powered email understanding
- 📝 Extract patient information
- 🎯 Detect patient intent
- 📅 Check doctor availability with Google Calendar
- 🏥 Answer insurance questions
- 💊 Handle prescription refill requests
- 📋 Support appointment booking, cancellation, and rescheduling
- 📄 Generate professional AI email replies
- ✍️ Create Gmail drafts instead of sending automatically
- 👨‍⚕️ Human approval before sending
- 📊 Log patient requests to Google Sheets or a database
- 🔔 Notify receptionists through Slack, Teams, Discord, WhatsApp, or Email

---

# Supported Patient Requests

- Appointment Booking
- Appointment Cancellation
- Appointment Rescheduling
- Insurance Questions
- Prescription Refill
- Lab Results
- Medical Records
- Billing Questions
- General Clinic Questions
- Emergency Requests

---

# Workflow Overview

```text
Patient Sends Email
        │
        ▼
Gmail Trigger
        │
        ▼
Read Email
        │
        ▼
Clean HTML
        │
        ▼
AI Analysis
        │
        ▼
Extract Patient Data
        │
        ▼
Intent Detection
        │
        ▼
Switch Node
 ┌────────┼────────┬────────┬────────┐
 │        │        │        │
 ▼        ▼        ▼        ▼
Appointment Insurance Billing Prescription
 │
 ▼
Google Calendar
 │
 ▼
Knowledge Base
 │
 ▼
Generate AI Reply
 │
 ▼
Human Approval
 │
 ▼
Create Gmail Draft
 │
 ▼
Update Database
 │
 ▼
Notify Receptionist
 │
 ▼
Workflow Complete
```

---

# Step-by-Step Workflow

## Step 1 — Patient Sends an Email

Patients contact the clinic through Gmail.

Examples:

- Book an appointment
- Ask about insurance
- Cancel an appointment
- Request a prescription refill
- Ask clinic hours
- Request lab results

Example:

```
Subject: Appointment Request

Hello,

I'd like to book an appointment with Dr. Sarah next Tuesday afternoon.

Also, do you accept Blue Cross insurance?

Thanks,
John Smith
```

---

## Step 2 — Gmail Trigger

The workflow automatically starts whenever a new email arrives.

Outputs:

- Sender
- Subject
- Email Body
- Attachments
- Timestamp

---

## Step 3 — Clean Email

The workflow removes:

- HTML
- Email signatures
- Previous replies
- Forwarded content

Result:

```
Hello,

I'd like to book an appointment with Dr. Sarah next Tuesday afternoon.

Also, do you accept Blue Cross insurance?
```

---

## Step 4 — AI Analysis

The AI extracts structured information.

Example:

```json
{
  "patient_name": "John Smith",
  "intent": "appointment",
  "doctor": "Dr. Sarah",
  "date": "Next Tuesday",
  "time": "Afternoon",
  "insurance": "Blue Cross",
  "urgency": "Normal",
  "summary": "Patient wants to schedule an appointment and asks about insurance."
}
```

---

## Step 5 — Intent Detection

Supported intents:

- Appointment Booking
- Appointment Cancellation
- Appointment Reschedule
- Insurance Question
- Prescription Refill
- Medical Records
- Lab Results
- Billing Question
- General Question
- Emergency

---

## Step 6 — Route to the Correct Workflow

Depending on the detected intent:

### Appointment

- Check Google Calendar
- Suggest available time slots

### Insurance

- Search clinic policies
- Answer coverage questions

### Prescription

- Forward request to doctor/pharmacist

### Billing

- Search billing FAQ

### Emergency

- Notify staff immediately
- Mark email as high priority

---

## Step 7 — External Integrations

Supported integrations:

- Google Calendar
- Gmail
- Google Sheets
- Clinic Database
- CRM
- Knowledge Base
- Patient Database

Example:

```
Doctor Available?

YES
 ↓
Offer Appointment

NO
 ↓
Suggest Alternative Times
```

---

## Step 8 — AI Reply Generation

Example reply:

```
Hello John,

Thank you for contacting ABC Medical Clinic.

Dr. Sarah is available next Tuesday at 2:30 PM.

We also accept Blue Cross insurance.

Please reply to confirm your appointment.

Kind regards,

Reception Team
```

---

## Step 9 — Human Approval (Optional)

Before sending:

- Review AI Summary
- Edit Response
- Approve
- Reject

---

## Step 10 — Gmail Draft

Instead of automatically sending emails, the workflow creates a Gmail draft for review.

---

## Step 11 — Update Records

Example Google Sheet

| Patient | Intent | Status |
|----------|--------|--------|
| John Smith | Appointment | Pending |

---

## Step 12 — Notifications

Optional notifications:

- Slack
- Microsoft Teams
- Discord
- WhatsApp
- Email

Example:

```
New Appointment Request

Patient:
John Smith

Doctor:
Dr. Sarah

Status:
Draft Created
```

---

# Technologies Used

- n8n
- OpenAI
- Ollama
- Gmail API
- Google Calendar API
- Google Sheets
- AI Agents

---

# Benefits

✅ Saves receptionist time

✅ Responds to patients faster

✅ Consistent AI-generated replies

✅ Human approval before sending

✅ Reduces missed emails

✅ Easy to extend for SMS, WhatsApp, Voice AI, or Patient Portals

---

# Future Improvements

- Voice AI (Vapi)
- WhatsApp Automation
- SMS Notifications
- Patient Portal Integration
- EHR Integration
- CRM Synchronization
- Automatic Appointment Confirmation
- AI Phone Receptionist

---

# Demo

*(Add screenshots or GIFs here)*

---

# Installation

1. Import the workflow into n8n.
2. Configure Gmail credentials.
3. Connect OpenAI or Ollama.
4. Configure Google Calendar.
5. Connect Google Sheets (optional).
6. Activate the workflow.

---

# Requirements

- n8n
- Gmail Account
- Google Calendar
- OpenAI API Key or Ollama
- Google Sheets (optional)

---

# License

MIT License

---

## ⭐ Support

If you found this project useful, consider giving it a ⭐ on GitHub.

It helps others discover the project and supports future AI automation tutorials.