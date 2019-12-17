---
id: api
title: JavaScript API
sidebar_label: JavaScript API
---

This JavaScript browser API allows web monetized websites to request small
payments from users. Payments are sent to the website by the user's [Web
Monetization provider](/#providers).

> No files are needed to use this API.

This API is injected by any browser or extension that supports Web Monetization.

> This document represents the current API implementation. Note that an
[unofficial draft specification](../specification.html) suitable for the W3C
standards track is being developed and should **NOT** be used as a reference for
current implementations.

#### Assumptions

* Your website is [monetized](./getting-started).
* The user visiting your site has an account with a Web Monetization provider.
* The user's browser supports Web Monetization natively or through an extension.

## Document.monetization

The browser exposes the `document.monetization` DOM object that implements
[`EventTarget`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget)
and has a read-only `state` property. The object allows you to track Web
Monetization events and see whether the user visiting your page is web monetized.

```ts
document.monetization: EventTarget
document.monetization.state: 'stopped' | 'pending' | 'started'
```

## States

Check the value of `document.monetization.state` to see if a user is web
monetized.

### `undefined`

Until Web Monetization is built into browsers, the `state` will be `undefined`
unless a polyfill is installed (e.g. a Web Monetization provider's browser
extension).

```ts
document.monetization === undefined
```

### `stopped`

The extension is capable of Web Monetization but is not currently sending
micropayments, nor trying to.

```ts
document.monetization && document.monetization.state === 'stopped'
```

If your site is adding a `meta` tag dynamically, then the `state` begins in
`stopped`. It transitions to `pending` after the tag is added.

### `pending`

The extension is trying to send payments but has yet to send the first non-zero
micropayment.

```ts
document.monetization && document.monetization.state === 'pending'
```

If the extension recognizes your site's `meta` tag, then the `state` begins
in `pending`.

### `started`

The extension is currently sending micropayments.

```ts
document.monetization && document.monetization.state === 'started'
```

It can take a few seconds to change from `pending` to `started`. The events
described below can be used to listen for a change in `document.monetization.state`.

## Browser events

### `monetizationpending`

Determine when Web Monetization is enabled by adding an event listener for
`monetizationpending` to `document.monetization`.

#### Event listener

```ts
function pendingEventHandler (event) {
  console.log(event)
}

document
  .monetization
  .addEventListener('monetizationpending', pendingEventHandler)
```

#### Event object

| Field | Value | Details |
| ------ | ----- | ------ |
| paymentPointer | String | Your payment account URL. The same value is used as the `content` in your `<meta>` tag.|
| requestId | String | This value is identical to the session ID/monetization ID (UUID v4) generated by the user agent (see [Flow](./explainer#flow)).|

#### Example event object

```ts
{
  detail: {
    paymentPointer: "$$wallet.example.com/alice",
    requestId: "ec4f9dec-0ba4-4029-8f6a-29dc21f2e0ce"
  }
}
```

### `monetizationstart`

Determine when Web Monetization has started actively paying by adding an event
listener for `monetizationstart` to `document.monetization`.

#### Event listener

```ts
function startEventHandler (event) {
  console.log(event)
}

document
  .monetization
  .addEventListener('monetizationstart', startEventHandler)
```

#### Event object

| Field | Value | Details |
| ------ | ----- | ------ |
| paymentPointer | String | Your payment account URL. The same value is used as the `content` in your `<meta>` tag.|
| requestId | String | This value is identical to the session ID/monetization ID (UUID v4) generated by the user agent (see [Flow](./explainer#flow)).|

#### Example event object

```ts
{
  detail: {
    paymentPointer: "$$wallet.example.com/alice",
    requestId: "ec4f9dec-0ba4-4029-8f6a-29dc21f2e0ce"
  }
}
```

### `monetizationstop`

Determine when Web Monetization has stopped by adding an event listener for
`monetizationstop` to `document.monetization`.

#### Event listener

```ts
function stopEventHandler (event) {
  console.log(event)
}

document
  .monetization
  .addEventListener('monetizationstop', stopEventHandler)
```

#### Event object

| Field | Value | Details |
| ------ | ----- | ------ |
| paymentPointer | String | The payment account URL for the Web Monetization `meta` tag at the time when payment has stopped. |
| requestId | String | The session ID/monetization ID (UUID v4) [generated by the browser](./explainer#flow) for the Web Monetization `meta` tag at the time when payment has stopped. |
| finalized | Boolean | When `true`, the monetization tag has been removed or the `paymentPointer` changed. No more events with this `requestId` are expected.

#### Example event object

```ts
{
  detail: {
    paymentPointer: "$$wallet.example.com/alice",
    requestId: "ec4f9dec-0ba4-4029-8f6a-29dc21f2e0ce",
    finalized: false
  }
}
```

### `monetizationprogress`

Determine the current status of the payment stream by adding an event listener
for `monetizationprogress` to `document.monetization`.

#### Event listener

```ts
function progressEventHandler (event) {
  console.log(event)
}

document
  .monetization
  .addEventListener('monetizationprogress', progressEventHandler)
```

#### Event object

| Field | Value | Details |
| ----- | ----- | ------- |
| paymentPointer | String | Your payment account URL. The same value is used as the `content` in your `<meta>` tag.|
| requestId | String | This value is identical to the session ID/monetization ID (UUID v4) generated by the user agent (see [Flow](./explainer#flow)).|
|amount | String | The destination amount received as specified in the Interledger protocol (ILP) packet. |
|assetCode | String | The code (typically three characters) identifying the amount's unit. A unit, for example, could be a currency (USD, XRP). |
|assetScale | Number | The number of places past the decimal for the amount. For example, if you have USD with an asset scale of two, then the minimum divisible unit is cents. |

#### Example event object

In this example, the total amount of USD received is equal to 7567 × 10^-2
(75.67).

```ts
{
  detail: {
    paymentPointer: "$wallet.example.com/alice",
    requestId: "ec4f9dec-0ba4-4029-8f6a-29dc21f2e0ce"
    amount: "7567",
    assetCode: "USD",
    assetScale: 2
  }
}
```

## HTTP headers

### `Web-Monetization-Id`

The `Web-Monetization-Id` header contains the monetization ID (UUID v4)
generated by the browser. The monetization ID is identical to the session ID and
the `requestId` in the `monetizationstart` browser event.

> The header **MUST ALWAYS** be sent on SPSP queries for Web Monetization and
**MUST** be a UUID v4.

#### Example header

```http
Web-Monetization-Id: dcd479ad-7d8d-4210-956a-13c14b8c67eb
```