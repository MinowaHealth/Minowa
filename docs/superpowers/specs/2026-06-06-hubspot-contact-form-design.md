# HubSpot Contact Form — Design

**Date:** 2026-06-06
**Project:** minowa.ai landing page
**Status:** Approved

## Summary

Add a contact form to the minowa.ai landing page that submits to HubSpot via
the public Forms Submission API. The site is a single static `index.html` with
inline CSS and no backend, so the form submits client-side with `fetch()`.

## Constraints

- **No backend.** Static HTML only (served in part for Apple Developer domain
  verification). All submission logic runs client-side.
- **Single-file.** Keep everything in `index.html`: markup, a `<style>` block
  extending existing CSS variables, and a `<script>` block. No build step, no
  dependencies.
- **No third-party JS.** Do not use HubSpot's embed script; it would clash with
  the hand-tuned design.

## Integration approach

Custom HTML form posting JSON to HubSpot's public Forms Submission API:

```
POST https://api.hsforms.com/submissions/v3/integration/submit/{portalId}/{formGuid}
Content-Type: application/json
```

This endpoint is public (no API key/secret). The Form GUID is not a secret, so
it is safe to embed in client-side JS. Because the endpoint requires a JSON
body, a plain no-JS `<form action>` POST cannot reach it — `fetch()` is
required. The retained mailing address serves as the no-JS fallback.

## Fields

| UI field | Type  | HubSpot property | Required |
|----------|-------|------------------|----------|
| Name     | text  | `firstname`      | yes      |
| Email    | email | `email`          | yes      |

A free-text *Message* field is intentionally omitted (per user choice). It is a
one-block addition later if desired (textarea → a HubSpot property such as
`message`).

## Placement

Within the existing **Contact** section:

- **Replace** the "Email" and "General inquiries" rows with the form.
- **Keep** the mailing address below the form unchanged.

## Markup / structure

- Short intro line above the form.
- `<form>` with:
  - `<label>` + `<input type="text">` for Name (`required`).
  - `<label>` + `<input type="email">` for Email (`required`).
  - Visually-hidden honeypot field (`bot-field`) for spam filtering.
  - Submit `<button>` styled with `--accent`.
  - A status region (`aria-live="polite"`) for success/error messages.

## Styling

Extend the existing CSS using current variables (`--ink`, `--ink-soft`,
`--rule`, `--accent`, `--accent-ink`, `--paper`). Inputs use the `--rule`
border and `--paper` background; the submit button uses `--accent` with white
text. Respect the existing 480px mobile breakpoint.

## Data flow

On submit:

1. `preventDefault()`.
2. If the honeypot field is non-empty → silently no-op (treat as success-ish,
   do nothing / no network call).
3. Client validation: both fields non-empty; email matches a basic pattern.
   On failure → show inline error, focus first invalid field, stop.
4. Disable the button, set its label to "Sending…".
5. `fetch()` POST JSON:

   ```json
   {
     "fields": [
       { "name": "firstname", "value": "<name>" },
       { "name": "email", "value": "<email>" }
     ],
     "context": { "pageUri": "<location.href>", "pageName": "<document.title>" }
   }
   ```

6. On HTTP 200 → replace the form with an inline confirmation:
   "Thanks — we'll be in touch."
7. On non-200 or network error → re-enable the button, restore its label, show
   an inline error message ("Something went wrong — please email us at …" with
   the mailing address / a mailto as fallback).

## Configuration

Two clearly-marked JS constants with placeholder values:

```js
const HUBSPOT_PORTAL_ID = "YOUR_PORTAL_ID";   // e.g. "12345678"
const HUBSPOT_FORM_GUID  = "YOUR_FORM_GUID";  // e.g. "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

### How to obtain the IDs (HubSpot account exists, form not yet created)

1. In HubSpot, go to **Marketing → Forms → Create form**.
2. Choose an **Embedded** form; add **First name** and **Email** fields
   (Email is required by HubSpot).
3. Publish the form.
4. **Portal ID (Hub ID):** account menu / Settings → Account Information, or
   visible in the form embed code. It is the numeric ID.
5. **Form GUID:** in the form's **Share/Embed** code, the
   `.../submit/{portalId}/{formGuid}` URL — the GUID is the second value.
6. Paste both into the constants above.

## Accessibility

- `<label>` associated with each input.
- `required` attributes for native validation backup.
- `aria-describedby` linking inputs to the status region.
- `aria-live="polite"` on the status region so screen readers announce
  success/error.
- Button disabled + "Sending…" during the request.

## Verification

- Validate the HTML structure.
- With real Portal ID + Form GUID pasted in, submit the form and confirm the
  contact appears in HubSpot under the form's submissions view.
- Confirm success and error states render correctly.

## Out of scope

- Message/textarea field (deferred).
- CAPTCHA / reCAPTCHA (honeypot only).
- Analytics tracking cookie (`hutk`) integration (optional future addition).
- Multi-page or multi-form support.
