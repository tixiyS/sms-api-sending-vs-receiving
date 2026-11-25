# sms-api-sending-vs-receiving
# What Is an SMS API? Sending vs Receiving, Explained

Search for "SMS API" and you'll get a wall of providers that send text messages: marketing blasts, delivery alerts, login codes. That's one kind of SMS API. There's a second kind that works the other way around: it gives you phone numbers and hands you the SMS that arrives on them.

If you pick the wrong kind, you'll spend a day reading documentation that can't do what you need. This guide explains both types, shows when each one makes sense, and walks through how a receiving API works in practice, using the [Tiger SMS](https://tiger-sms.com/?utm_source=github) API as the example.

## Contents

- [What Is an SMS API?](#what-is-an-sms-api)
- [Sending vs Receiving SMS APIs](#sending-vs-receiving-sms-apis)
- [When You Need an SMS Messaging API](#when-you-need-an-sms-messaging-api)
- [When You Need a Receiving API](#when-you-need-a-receiving-api)
- [How a Receiving SMS API Works](#how-a-receiving-sms-api-works)
- [SMS API Integration Checklist](#sms-api-integration-checklist)
- [FAQ](#faq)
- [Bottom Line](#bottom-line)

## What Is an SMS API?

An SMS API is an interface that lets your code work with text messages instead of a person tapping a phone. Your app sends an HTTP request, and the provider does the telecom work: routing messages through carriers, buying numbers, tracking delivery.

Most SMS APIs fall into one of two groups:

- **Sending APIs** push messages from your system to people's phones.
- **Receiving APIs** give you numbers that collect incoming messages, usually verification codes, and return them to your code.

## Sending vs Receiving SMS APIs

| | Sending (messaging) API | Receiving (verification number) API |
|---|---|---|
| **What it does** | Delivers your messages to users | Rents a number and returns the SMS it gets |
| **Typical use** | OTP for your users, alerts, reminders, campaigns | Testing signup flows, automating account verification |
| **Who owns the recipient** | Your customers | You, for the length of the activation |
| **Main operations** | Send, check delivery status | Get number, check status, cancel or finish |
| **Example providers** | Twilio, Vonage, Telnyx | Tiger SMS and similar activation services |

## When You Need an SMS Messaging API

Choose a sending API, often called an SMS messaging API, when your product talks to its own users:

- you send one-time passwords when people log in to your app;
- you notify customers about orders, bookings, or payments;
- you run reminder or marketing campaigns.

Here the phone numbers belong to your users. Your job is to get messages to them reliably and cheaply.

## When You Need a Receiving API

You need the opposite when your code must receive a verification code from someone else's service. This is where terms like *phone verification API*, *receive SMS API*, and *virtual number API* come in.

Common cases:

- **QA and test automation.** End-to-end tests that cover signup or login with a phone number need a real number and a real code, every time the pipeline runs.
- **Operations at scale.** Teams that register many accounts for legitimate work, such as regional storefronts or support profiles, need numbers they can request on demand.

The numbers matter too: many services reject VoIP lines, so an SMS verification API is only useful if its numbers pass. Tiger SMS, for example, provides non-VoIP numbers.

## How a Receiving SMS API Works

Receiving APIs revolve around an **activation**: one number, rented for one service, until the code arrives or you cancel. The Tiger SMS API is a good example of the flow.

### 1. Authenticate

Requests go to `https://api.tiger-sms.com/stubs/handler_api.php`, and your key is passed as the `api_key` query parameter. A missing or wrong key returns `BAD_KEY`.

### 2. Check balance and prices

`getBalance` returns your balance, for example `ACCESS_BALANCE:12.3400`, or JSON if you add `format=json`.

`getPrices` returns JSON grouped by country and then by service, with the cost and the number of numbers available. Check it before you buy, so your code knows where stock exists.

### 3. Get a number

Call `getNumber` with the service and country you need. Services use short codes, such as `tg` for Telegram and `wa` for WhatsApp. Countries use numeric IDs, such as `187` for the United States.

```http
GET https://api.tiger-sms.com/stubs/handler_api.php?action=getNumber&api_key=YOUR_KEY&service=tg&country=187
```

```text
ACCESS_NUMBER:557860099:18734721259
```

The response holds the activation ID and the phone number. If nothing is available, you'll get errors such as `NO_NUMBERS` or `NO_BALANCE`, which your code should handle rather than retry blindly.

There's also `getNumberV2`, which returns JSON with the activation ID, phone number, activation cost, currency, and the time the activation ends.

### 4. Wait for the code

Use the number in the service you're testing, then poll `getStatus` with the activation ID:

| Response | Meaning |
|---|---|
| `STATUS_WAIT_CODE` | Nothing yet |
| `STATUS_OK:852508` | The code has arrived |
| `ACCESS_CANCEL` | The activation was cancelled |

Don't poll aggressively. The documentation notes that checking more often than every few seconds just returns the same answer, and a rate limit responds with `Too Many Requests` and a `Retry-After` header. If you'd rather not poll at all, the Tiger SMS API also offers an incoming SMS webhook, which you subscribe to in your account.

`getStatusV2` gives the same information as JSON, including the full SMS text and the code.

### 5. Close the activation

`setStatus` moves the activation through its states with numeric values: `1`, `3`, `6`, and `8`. Two are worth knowing first:

- `3` asks for another SMS on the same activation;
- `8` cancels the activation.

> [!NOTE]
> Cancelling has two edge cases. If the SMS already arrived, the API returns the code instead of cancelling. And if you cancel too soon after buying, you'll get `EARLY_CANCEL_DENIED`.

## SMS API Integration Checklist

Before you ship, run through these points:

- [ ] **Keep the key secret.** It goes in the query string, so never expose it in front-end code or public logs.
- [ ] **Handle every error string.** `NO_NUMBERS`, `NO_BALANCE`, `BAD_SERVICE`, and `BAD_COUNTRY` all need a path in your code.
- [ ] **Respect the rate limit.** Poll every few seconds, read `Retry-After`, or use the webhook.
- [ ] **Cancel what you don't use.** Release numbers that never get a code so your balance isn't tied up.
- [ ] **Test carefully.** Tiger SMS publishes an OpenAPI spec and a Postman collection for setup. The live request tester on the docs page sends real requests, which may charge your balance or change a real activation.
- [ ] **Plan for SMS-Activate code.** The Tiger SMS API is compatible with the SMS-Activate request format, so scripts written for SMS-Activate can usually be moved with small changes.

## FAQ

<details>
<summary><b>Is an SMS API the same as an SMS gateway?</b></summary>

They overlap. "Gateway" usually describes the infrastructure that connects to carriers, while "API" is the interface your code calls. Many providers use both terms.

</details>

<details>
<summary><b>Can one API both send and receive?</b></summary>

Some messaging providers let you receive replies on numbers you rent long-term. Activation services like Tiger SMS focus on receiving verification codes and don't send messages to users.

</details>

<details>
<summary><b>What's the difference between a phone verification API and a sending OTP API?</b></summary>

A sending OTP API delivers codes to your users. A phone verification API of the receiving kind gives you a number and returns the code another service sends to it.

</details>

<details>
<summary><b>Do I need a webhook?</b></summary>

No. Polling works fine at a sensible interval. A webhook just delivers the SMS the moment it arrives, without repeated requests.

</details>

## Bottom Line

An SMS API either sends messages to your users or receives them on numbers you rent. If you're delivering OTPs and alerts, look at a messaging provider. If your code needs to receive verification codes, for testing or automation, you need a receiving API built around activations. The [Tiger SMS API](https://tiger-sms.com) covers that flow end to end: prices and stock, non-VoIP numbers on demand, status polling or a webhook, cancellations, and a request format compatible with SMS-Activate.
