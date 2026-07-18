# AI WhatsApp Appointment Booking Bot

A conversational AI-powered appointment booking system built with **Meta WhatsApp Cloud API**, **n8n**, **Ollama**, **Google Calendar**, and **Gmail**.

Users can book appointments naturally through WhatsApp without filling forms or following rigid menus. The AI understands natural language, checks calendar availability, books appointments, and sends confirmation emails.

---

# Features

- 🤖 Natural language conversations using Ollama
- 💬 WhatsApp Business integration via Meta Cloud API
- 📅 Google Calendar availability checking
- 📆 Automatic appointment booking
- 📧 Confirmation email sent after booking
- 🧠 Conversation memory throughout the booking process
- 🔄 Suggests alternative slots if requested time is unavailable
- ✏️ Supports rescheduling appointments
- ❌ Supports appointment cancellation
- 🚀 Fully automated using n8n

---

# Tech Stack

| Component | Technology |
|------------|------------|
| Messaging | Meta WhatsApp Cloud API |
| Workflow Automation | n8n |
| AI Agent | Ollama |
| Calendar | Google Calendar API |
| Email | Gmail SMTP / Gmail API |
| Memory | n8n Data Store / Redis |
| LLM | Llama 3, Mistral, Gemma, etc. |

---

# System Architecture

```text
                User
                  │
                  ▼
      Meta WhatsApp Cloud API
                  │
                  ▼
               Webhook
                  │
                  ▼
                 n8n
                  │
      ┌───────────┼────────────┐
      │           │            │
      ▼           ▼            ▼
   Ollama    Google Calendar   Gmail
      │           │            │
      └───────────┴────────────┘
                  │
                  ▼
             WhatsApp Reply
```

---

# Project Flow

```text
User sends message

↓

WhatsApp Webhook

↓

Load Conversation Memory

↓

Ollama AI Agent

↓

Determine User Intent

↓

Need Calendar Check?

├── No
│      ↓
│  Continue Conversation
│
└── Yes
       ↓
Google Calendar

↓

Available?

├── Yes
│      ↓
│ Ask Name
│
│ Ask Email
│
│ Create Event
│
│ Send Confirmation Email
│
│ Reply on WhatsApp
│
└── No
       ↓
Suggest Top 3 Available Slots
```

---

# Booking Flow

## Step 1

User

> Hi, I'd like to book an appointment.

AI

> Sure! What day would you like to book?

---

## Step 2

User

> Tomorrow afternoon

n8n checks Google Calendar.

Available slots:

- 2:00 PM
- 3:00 PM
- 4:30 PM

AI

> These times are available tomorrow afternoon:
>
> • 2:00 PM
> • 3:00 PM
> • 4:30 PM
>
> Which one would you like?

---

## Step 3

User

> 3 PM

n8n verifies availability.

Available ✅

AI

> Great! What's your full name?

---

## Step 4

User

> John Smith

AI

> Thanks John! What's your email address?

---

## Step 5

User

> john@gmail.com

n8n

- Creates Google Calendar Event
- Sends Confirmation Email

AI

> ✅ Your appointment has been booked successfully.
>
> I've also sent a confirmation email to john@gmail.com.

---

# Alternative Slot Flow

User

> Friday at 4 PM

Google Calendar

Busy ❌

AI

> That time isn't available.
>
> Here are the closest available times:
>
> • 3:30 PM
> • 4:30 PM
> • 5:00 PM

---

# Reschedule Flow

User

> I want to reschedule my appointment.

Workflow

1. Find user's appointment
2. Show available slots
3. User selects new slot
4. Update Google Calendar
5. Send updated confirmation email

---

# Cancel Appointment Flow

User

> Cancel my appointment.

Workflow

1. Find appointment
2. Delete Google Calendar event
3. Send cancellation email
4. Confirm cancellation on WhatsApp

---

# Conversation Memory

The system stores conversation state for each WhatsApp user.

Example:

```json
{
  "phone": "+923001234567",
  "intent": "book_appointment",
  "date": "2026-07-20",
  "selected_slot": "15:00",
  "name": "John Smith",
  "email": "john@gmail.com",
  "status": "collecting_email"
}
```

Memory can be stored in:

- n8n Data Store
- Redis
- Database

---

# Ollama Responsibilities

The AI is responsible for:

- Detecting booking intent
- Understanding natural language
- Asking only one question at a time
- Tracking missing information
- Producing friendly responses

The AI does **not**:

- Access Google Calendar
- Create appointments
- Send emails

Those tasks are handled by n8n.

---

# n8n Responsibilities

n8n orchestrates the entire workflow.

Responsibilities include:

- Receive WhatsApp messages
- Maintain conversation state
- Call Ollama
- Check Google Calendar
- Create calendar events
- Send emails
- Reply to WhatsApp users

---

# Google Calendar Workflow

## Check Availability

- Receive requested date/time
- Query Google Calendar
- Detect busy slots
- Generate available slots
- Return top three matching times

## Create Appointment

Create a calendar event with:

- Customer Name
- Email Address
- Appointment Date
- Appointment Time
- Duration
- Event Description

---

# Email Confirmation

Subject

```
Appointment Confirmed
```

Example

```
Hello John,

Your appointment has been confirmed.

Date:
Monday, July 20

Time:
3:00 PM

Thank you.

We look forward to seeing you.
```

---

# AI System Prompt

The Ollama assistant should:

- Be friendly and conversational
- Ask only one question at a time
- Understand natural language dates and times
- Never request information already provided
- Request only missing information
- Suggest three alternatives when a slot is unavailable
- Keep responses short and natural
- Never expose technical implementation details

---

# Future Improvements

- Multi-staff scheduling
- Multiple calendars
- Buffer time between appointments
- Working hours configuration
- Holiday support
- Timezone support
- Appointment reminders
- Follow-up messages
- Payment integration
- CRM integration
- Admin dashboard
- Analytics and reporting

---

# License

This project is provided for educational and demonstration purposes.