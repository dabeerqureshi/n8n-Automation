# AI Email Auto Reply Workflow

This workflow automatically handles incoming Gmail messages using n8n, Ollama, and Gmail. When a new email is received, the workflow extracts the email content, sends it to a local AI model running with Ollama to generate a professional reply, creates a draft response in Gmail, and finally marks the original email as read.

This automation helps save time by preparing AI-generated replies while still allowing you to review them before sending. Since Ollama runs locally, your email content remains on your own machine, making it a privacy-friendly solution without relying on cloud AI services.

## Workflow Steps

1. Detect new emails using the Gmail Trigger.
2. Extract the sender, subject, body, thread ID, and message ID using a Code node.
3. Generate a context-aware email reply with an AI Agent powered by Ollama.
4. Create a Gmail draft containing the AI-generated response in the same email thread.
5. Mark the original email as read to keep your inbox organized.

## Technologies Used

- n8n
- Gmail
- Ollama
- AI Agent Node
- JavaScript (Code Node)

This workflow is ideal for personal inbox management, customer support, freelancers, business owners, and anyone looking to automate repetitive email responses while maintaining full control over the final message.