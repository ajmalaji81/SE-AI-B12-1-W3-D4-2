# Slack Recipe Product Finder (n8n Workflow)

An automated event-driven workflow built in [n8n](https://n8n.io/) that listens for recipe queries in a designated Slack channel, generates an itemized shopping list and preparation summary using OpenAI's `gpt-4o-mini` via LangChain, normalizes the response using JavaScript, and posts the results back to Slack.

---

## 🏗 Workflow Architecture

```
┌──────────────────────────┐
│       Slack Trigger      │  Listens for incoming messages in Slack channel
└─────────────┬────────────┘
              │
              ▼
┌──────────────────────────┐
│        Edit Fields       │  Extracts and maps raw message text to `recipe_request`
└─────────────┬────────────┘
              │
              ▼
┌──────────────────────────┐      ┌──────────────────────────┐
│     Basic LLM Chain      │ ◄─── │    OpenAI Chat Model     │
│   (Prompt & Chain)       │      │      (gpt-4o-mini)       │
└─────────────┬────────────┘      └──────────────────────────┘
              │
              ▼
┌──────────────────────────┐
│    Code in JavaScript    │  Extracts and normalizes text from LLM response
└─────────────┬────────────┘
              │
              ▼
┌──────────────────────────┐
│      Send a Message      │  Publishes structured shopping list back to Slack
└──────────────────────────┘
```

---

## ⚡ Features

- **Real-time Event Handling:** Listens directly for channel activity using n8n's Slack Trigger.
- **Structured AI Prompting:** Instructs `gpt-4o-mini` to extract a succinct preparation overview paired with a single-ingredient-per-line shopping list with precise measurements.
- **Fail-Safe Output Parsing:** Custom JavaScript fallback handles multiple LangChain output structures (`text`, `response`, or `output`).
- **Autonomous Channel Responses:** Returns the generated list straight to the channel without requiring external orchestration.

---

## 📋 Prerequisites & Credentials

Ensure you have the following credentials ready in your n8n workspace:

### 1. Slack OAuth / Bot Token (`slackApi`)
- A Slack App configured under [Slack API Apps](https://api.slack.com/apps).
- **Bot Token Scopes (`Bot User OAuth Token`):**
  - `channels:history` (read messages in public channels)
  - `groups:history` (optional: read messages in private channels)
  - `chat:write` (post messages as the bot)
  - `channels:read` (resolve channel names/IDs)
- Event Subscriptions enabled pointing to your n8n production webhook URL (or managed via n8n's native Slack App credentials).

### 2. OpenAI API Credential (`openAiApi`)
- Valid OpenAI API key with access to `gpt-4o-mini`.

---

## 🔍 Node Breakdown

| Node Name | Node Type | Version | Purpose |
| :--- | :--- | :--- | :--- |
| **Slack Trigger** | `n8n-nodes-base.slackTrigger` | 1 | Listens for message events on channel `recipe-test` (`C0CCGD1TS15`). |
| **Edit Fields** | `n8n-nodes-base.set` | 3.5 | Maps `$json.text` to the standardized parameter `recipe_request`. |
| **OpenAI Chat Model** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | 1.3 | Configures the underlying language model (`gpt-4o-mini`). |
| **Basic LLM Chain** | `@n8n/n8n-nodes-langchain.chainLlm` | 1.9 | Prompts the model for preparation instructions and line-delimited shopping lists. |
| **Code in JavaScript** | `n8n-nodes-base.code` | 2 | Runs fallback logic: `const result = $json.text ?? $json.response ?? $json.output ?? '';`. |
| **Send a message** | `n8n-nodes-base.slack` | 2.7 | Posts the parsed result (`{{ $json.ingredients }}`) back to the Slack channel. |

---

## 🚀 Installation & Setup

1. **Import the Workflow:**
   - In your n8n canvas, select **Add Workflow** > click the three dots (`...`) in the top-right corner.
   - Choose **Import from File** or **Import from URL** and supply the workflow JSON.

2. **Configure Credentials:**
   - Attach your configured **Slack API** credential to the **Slack Trigger** and **Send a message** nodes.
   - Attach your **OpenAI API** credential to the **OpenAI Chat Model** sub-node.

3. **Set the Target Channel:**
   - Open **Slack Trigger** and select the channel you want to monitor.
   - Open **Send a message** and set the target channel to match your listener channel (or map dynamically using `={{ $('Slack Trigger').item.json.channel }}`).

4. **Activate the Workflow:**
   - Toggle the switch from **Inactive** to **Active** at the top right of the workflow editor.

---

## 💬 Usage & Example

1. **User asks in Slack:**
   > *"Can you give me a recipe for classic Spaghetti Carbonara?"*

2. **Bot replies:**
   ```text
   Short Preparation:
   Boil pasta until al dente. Crisp guanciale in a pan. Whisk egg yolks with Pecorino Romano and black pepper. Combine warm pasta with guanciale and egg mixture off heat to create a creamy sauce.

   Shopping List:
   400g Spaghetti
   150g Guanciale (or Pancetta)
   4 large Egg yolks
   1 whole Egg
   100g Pecorino Romano cheese (freshly grated)
   Freshly cracked Black Pepper
   Salt (for pasta water)
   ```

---

## 🛡 Recommended Production Tweaks

- **Loop Prevention:** Check `$json.user` in an `If` node to make sure the message author is not the bot's own user ID.
- **Threaded Replies:** Pass `{{ $('Slack Trigger').item.json.ts }}` to the Slack node's `Thread TS` field to respond in the message thread instead of cluttering the channel root.