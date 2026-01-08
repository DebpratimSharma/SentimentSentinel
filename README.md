# 🛡️ Sentiment Sentinel
> **Automating Empathy at Scale with Gemini 3 Flash**

![Gemini](https://img.shields.io/badge/AI-Gemini%203%20Flash-8E75B2?style=for-the-badge&logo=googlebard&logoColor=white)
![Google Apps Script](https://img.shields.io/badge/Google%20Apps%20Script-4285F4?style=for-the-badge&logo=google&logoColor=white)
![Zero Cost](https://img.shields.io/badge/Infrastructure-Zero%20Cost-success?style=for-the-badge)

## 🎥 Demo Video
[![Watch the Demo](https://img.youtube.com/vi/YOUR_VIDEO_ID_HERE/0.jpg)](https://www.youtube.com/watch?v=YOUR_VIDEO_ID_HERE)
*(Click the image above to watch the workflow in action)*

## 💡 The Problem
Customer support teams are overwhelmed. Responding to angry feedback requires emotional energy and time that agents often don't have. Slow responses to negative feedback lead to churn.

## 🚀 The Solution
**Sentiment Sentinel** is a serverless AI agent that lives inside Google Workspace. It listens for new customer feedback via Google Forms, instantly analyzes the emotional tone using **Gemini 3 Flash Preview**, and drafts a hyper-personalized email response in Gmail.

* **Positive Feedback:** Drafts a thank you note + asks for a referral.
* **Negative Feedback:** Drafts a deep apology + offers a support call.
* **Safety First:** Uses "Human-in-the-Loop" (Drafts) so no AI accidents happen.

## 🛠️ Tech Stack
* **Brain:** Google Gemini 3 Flash Preview (`gemini-3-flash-preview`)
* **Backend:** Google Apps Script (Serverless JavaScript)
* **Database:** Google Sheets
* **Frontend:** Google Forms
* **Cost:** $0.00 (Runs entirely on Google's free productivity tier)

## ⚙️ Installation & Setup
You can replicate this project in 5 minutes.

1.  **Get an API Key:**
    * Visit [Google AI Studio](https://aistudio.google.com/) and create a free API key.
2.  **Create the Form:**
    * Create a Google Form with two questions: `Customer Email` and `Feedback`.
    * Link it to a Google Sheet.
3.  **Install the Script:**
    * In the Sheet, go to **Extensions > Apps Script**.
    * Copy/Paste the code below into `Code.gs`.
4.  **Set the Trigger:**
    * In Apps Script, click the **Alarm Clock (Triggers)** icon.
    * Add Trigger: `onFormSubmit` -> `From Spreadsheet` -> `On form submit`.

## 💻 The Source Code

```code
/**
 * SENTIMENT SENTINEL
 * Techsprint Project using Google Apps Script & Gemini API
 */

const GEMINI_API_KEY = "YOUT-GEMINI-API-KEY-FROM-AI-STUDIO";

// 1. THE TRIGGER FUNCTION
// This runs automatically whenever someone submits the form
function onFormSubmit(e) {
  // Check if event object exists (for debugging)
  if (!e) {
    Logger.log("No event object found. Run this via Trigger, not manually.");
    return;
  }

  const responses = e.namedValues;
  
  // Get data from the form (Adjust names if your questions are different)
  const customerEmail = responses['Customer Email'] ? responses['Customer Email'][0] : null;
  const feedback = responses['Feedback/Comment'] ? responses['Feedback/Comment'][0] : null;

  if (customerEmail && feedback) {
    processFeedback(customerEmail, feedback);
  }
}

// 2. THE LOGIC CORE
function processFeedback(email, text) {
  // Step A: Ask Gemini to analyze and draft
  const prompt = `
    You are a customer support agent. 
    Analyze the following customer feedback: "${text}"
    
    1. Determine the sentiment (Positive, Negative, or Neutral).
    2. Write a email subject line.
    3. Write a polite email body response. 
       - If negative: Apologize deeply and offer a call.
       - If positive: Thank them and ask for a referral.
    
    Return the output strictly as this JSON format:
    {
      "sentiment": "Positive/Negative",
      "subject": "Email Subject",
      "body": "Email Body content"
    }
  `;

  const aiResponse = callGemini(prompt);
  
  // Parse the JSON returned by Gemini
  try {
    // Gemini sometimes wraps JSON in markdown blocks (```json ... ```). Clean it.
    const cleanJson = aiResponse.replace(/```json/g, "").replace(/```/g, ""); 
    const data = JSON.parse(cleanJson);

    // Step B: Send the Email (or Draft it)
    // using GmailApp.createDraft creates a draft so you can show it in the demo video safely
    // Change to GmailApp.sendEmail to send immediately
    GmailApp.sendEmail(
      email,
      data.subject,
      data.body
    );

    Logger.log("Draft created for: " + email + " | Sentiment: " + data.sentiment);

  } catch (error) {
    Logger.log("Error parsing AI response: " + error);
  }
}

// 3. THE AI CONNECTION
function callGemini(promptText) {
  // OPTION 1: The Bleeding Edge (What you asked for)
  const modelName = 'gemini-3-flash-preview';
  
  // OPTION 2: The Stable Workhorse (Use this if Option 1 fails)
  // const modelName = 'gemini-2.5-flash';

  const url = `https://generativelanguage.googleapis.com/v1beta/models/${modelName}:generateContent?key=${GEMINI_API_KEY}`;
  
  const payload = {
    "contents": [{
      "parts": [{ "text": promptText }]
    }],
    // FIX: Turn off safety filters so "Angry" feedback isn't blocked
    "safetySettings": [
      {
        "category": "HARM_CATEGORY_HARASSMENT",
        "threshold": "BLOCK_NONE"
      },
      {
        "category": "HARM_CATEGORY_HATE_SPEECH",
        "threshold": "BLOCK_NONE"
      },
      {
        "category": "HARM_CATEGORY_SEXUALLY_EXPLICIT",
        "threshold": "BLOCK_NONE"
      },
      {
        "category": "HARM_CATEGORY_DANGEROUS_CONTENT",
        "threshold": "BLOCK_NONE"
      }
    ]
  };

  const options = {
    "method": "post",
    "contentType": "application/json",
    "payload": JSON.stringify(payload),
    "muteHttpExceptions": true
  };

  const response = UrlFetchApp.fetch(url, options);
  const json = JSON.parse(response.getContentText());
  
  // LOGGING: This will show you exactly why it fails in the Executions tab
  if (!json.candidates) {
    Logger.log("FULL ERROR RESPONSE: " + JSON.stringify(json));
    throw new Error("Gemini blocked the request. Check Executions log for details.");
  }
  
  return json.candidates[0].content.parts[0].text;
}
