# AI Email Spam Detection Workflow

This workflow automatically monitors your Gmail inbox for new emails. When a new email is received, it extracts the subject and body, then sends the content to a local AI model running with Ollama to determine whether the email is spam. If the email is classified as spam, it is automatically moved to the Gmail Spam folder. If it is not spam, no action is taken and the email remains in the inbox.

## Workflow Steps

1. **Gmail Trigger**
   - Detects new incoming emails.

2. **Code Node**
   - Extracts the email subject and body.

3. **AI Agent (Ollama)**
   - Analyzes the email content and classifies it as either `spam` or `not_spam`.

4. **IF Node**
   - Checks the AI classification result.

5. **Gmail**
   - If the result is `spam`, moves the email to the Gmail Spam folder.

6. **End**
   - If the result is `not_spam`, the workflow ends without making any changes.

## Requirements

- n8n
- Gmail Account
- Gmail Credentials
- Ollama
- A supported local LLM (e.g. Llama 3, Mistral, Gemma)

## Use Cases

- Automatically filter spam emails.
- Keep your inbox organized.
- Reduce manual spam management.
- Run spam detection locally for improved privacy.