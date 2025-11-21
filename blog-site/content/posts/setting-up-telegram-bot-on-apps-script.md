---
date: '2025-07-22T23:24:14+05:00'
draft: false
title: 'Setting Up Telegram Bot on Apps Script'
tags: ["apps-script", "telegram-bot"]
---

Here is technical side of what I did on development on Apps Script. [main article](/posts/apps-script) 

```javascript
// Main webhook handler - receives all Telegram updates
function doPost(e) {
    try {
        if (!e || !e.postData) {
            return ContentService.createTextOutput('Invalid request').setMimeType(ContentService.MimeType.TEXT).setStatusCode(400);
        }

        const update = JSON.parse(e.postData.contents);

        if (update.message) {
            handleMessage(update.message);
        }

        // IMPORTANT: Always return OK with proper content type and mainly proper status code (200<=status<300) to prevent retries
        return ContentService.createTextOutput('OK').setMimeType(ContentService.MimeType.TEXT).setStatusCode(200);
    } catch (error) {
        console.error('Error in doPost:', error);
        // Even on error, return OK to prevent Telegram retries
        return ContentService.createTextOutput('OK').setMimeType(ContentService.MimeType.TEXT).setStatusCode(500);
    }
}


```
By the nature of apps script, `doPost` is used when the app received POST request. Telegram sends data via POST to webhook url. So we set up POST listener.

```javascript
function handleMessage(message) {
    try {
        // Ignore messages from bots or the bot itself
        if (message.from.is_bot) {
            return;
        }

        const text = message.text || '';
        const chatId = message.chat.id;

        echoMsg(message)

        // Handle commands
        if (text.startsWith('/')) {
            handleCommand(message, text);
            return;
        }

        // Process load data from groups/supergroups
        if (message.chat.type === 'group' || message.chat.type === 'supergroup') {
            processLoadMessage(message);
        }

    } catch (error) {
        console.error('Error handling message:', error);
    }
}
```
By business requirement we only must listen groups and supergroups

```javascript
function setWebhook() {
    const token = BOT_TOKEN;
    const url = WEB_APP_URL;

    UrlFetchApp.fetch(`https://api.telegram.org/bot${token}/setWebhook?url=${url}`);
}

// EMERGENCY: Stop webhook immediately
function deleteWebhook() {
    try {
        const url = `${TELEGRAM_URL}/deleteWebhook`;
        const response = UrlFetchApp.fetch(url);
        console.log('🛑 Webhook DELETED:', response.getContentText());
    } catch (error) {
        console.error('Error deleting webhook:', error);
    }
}

// Get webhook status
function getWebhookInfo() {
    try {
        const url = `${TELEGRAM_URL}/getWebhookInfo`;
        const response = UrlFetchApp.fetch(url);
        console.log('Webhook info:', response.getContentText());
    } catch (error) {
        console.error('Error getting webhook info:', error);
    }
}

```

Those are used to interact with telegram bot api to set up and remove webhook info. 

### How to get WEB_APP_URL?
Deploy the code with Deploy menu.
* open menu on the top right side of the screen.
* choose "New Deployment"
* Choose web app on select type menu
* Choose Anyone to "who has access"
* press deploy

if deployment goes successful, the app gives you deployment url, it is used as WEB_APP_URL. save it to some variable, and send request to telegram api via setWebhook function. 
On new deployment. open manage deployments, archive old deployments and deploy it again, take the deployment url, change the WEB_APP_URL variable, delete webhook and set webhook