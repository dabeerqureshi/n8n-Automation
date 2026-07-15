# AI Email Summarizer & Smart Draft Reply

This n8n workflow automatically monitors your Gmail inbox for new emails, summarizes each email using a local AI model running on Ollama, generates a professional reply, and saves the response as a Gmail draft for manual review. The workflow never sends emails automatically.

---

## Workflow Steps

### 1. Gmail Trigger
- Monitors Gmail for new incoming emails.
- Triggers the workflow whenever a new email is received.

### 2. Get Email Details
- Retrieves the email subject.
- Retrieves the sender's email address.
- Extracts the email body.

### 3. AI Email Summarizer (Ollama)
- Sends the email content to Ollama.
- Generates a concise summary highlighting the key points and action items.

### 4. AI Draft Reply Generator
- Uses the original email and its summary.
- Generates a professional, context-aware reply.
- Returns only the email body without signatures or formatting.

### 5. Create Gmail Draft
- Creates a new Gmail draft.
- Sets the recipient as the original sender.
- Prefixes the subject with **Re:**
- Inserts the AI-generated reply into the draft.

### 6. Workflow Complete
- The email has been summarized.
- A reply has been generated.
- The response is saved as a Gmail draft, ready for review and manual sending.

---

## Features

- Automatically monitors new Gmail emails
- AI-powered email summarization
- Professional draft reply generation
- Saves replies as Gmail drafts
- No automatic email sending
- Runs locally using Ollama
- Fully automated with n8n