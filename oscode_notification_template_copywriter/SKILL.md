---
name: notification_template_copywriter
description: Use when reviewing, drafting, or converting transactional notification templates for email and WhatsApp, especially mentor/student lifecycle messages. Produces polished copy plus implementation-ready template metadata such as snake_case names, variables, subject, body_text, body_html, and channel-specific notes.
---

# Notification Template Copywriter

## Goal

Review and produce notification templates that are clear, warm, implementation-ready, and suitable for both email and WhatsApp.

Use this skill for:
- Email template review
- WhatsApp template review
- Transactional lifecycle messages
- Mentor/student onboarding messages
- Template DB seed content
- Subject line and template naming
- Variable extraction

## Brand Voice

Write in a tone that is:
- Warm but not overly excited
- Clear and direct
- Respectful of uncertainty
- Useful, not salesy
- Human, founder-friendly, and lightweight

Avoid:
- Overpromising outcomes
- Excessive punctuation like `!!`
- Cute labels that may confuse the recipient
- Pushy prep instructions when approval/rejection is still possible
- Long paragraphs
- Corporate filler like “we value your association”

Prefer:
- “Welcome to {{brand_name}}”
- “There’s nothing else you need to do right now”
- “We’ll keep you posted”
- “If you need help, just reply here”
- “You can keep a few ideas handy”

## Email Style

Email templates should include:
- Clear subject
- Friendly greeting
- One direct purpose
- Short paragraphs
- Scannable next steps when needed
- Helpful CTA
- Footer/signature

Email structure:

1. Header  
   Use brand/logo if available in HTML layout. Do not repeat brand heavily in body.

2. Greeting  
   `Hi {{first_name}},`

3. Main message  
   State what happened and why the email was sent.

4. Next steps  
   Use numbered steps only if the user must complete multiple actions.

5. CTA  
   Use one primary CTA link.

6. Support line  
   Tell users how to get help.

7. Footer  
   Use either:
   - `Team {{brand_name}}`
   - or founder signature for high-touch onboarding:
     `Warm regards, Harsh, Founder, OSCode`

## WhatsApp Style

WhatsApp templates should be shorter than email.

Rules:
- Keep it concise
- Avoid HTML
- Avoid long signatures
- Avoid sending passwords or sensitive credentials unless explicitly required
- Use short numbered steps only when helpful
- Include one link if needed
- End with a simple support line

Preferred WhatsApp ending:
`If you need any help, just reply here.`

## Header And Footer Guidance

For email HTML:
- If the system has a shared email layout, return only content body HTML and mention that header/footer should come from the shared layout.
- If the template must include full HTML, include:
  - logo/header
  - body content
  - divider
  - footer/signature

Footer examples:

Founder-style:
```text
Warm regards,
Harsh
Founder, OSCode
```
Team-style:
```text
Best,
Team {{brand_name}}
```
WhatsApp footer:
```text
Team {{brand_name}}
```

## Template Naming
Return template names in `snake_case`.

Pattern:
```text
{audience_or_entity}_{event}_{status_or_purpose}_{channel}
```
Examples:
```text
mentor_application_submitted_success_email
mentor_application_submitted_success_whatsapp
mentor_application_approved_onboarding_email
mentor_application_approved_onboarding_whatsapp
```

## Variable Rules

Use clear snake_case variables.

Common variables:
```text
{{mentor_first_name}}
{{mentor_email}}
{{temporary_password}}
{{brand_name}}
{{sign_in_url}}
{{onboarding_video_url}}
{{support_email}}
```
Do not invent variables unless needed. Always list all variables used.

## Required Output Format

Always return implementation-ready output in this shape:
```yaml
name: template_name_in_snake_case
channel: email | whatsapp
locale: en
subject: Email subject or null
variables:
  - variable_one
  - variable_two
body_text: |
  Plain text template here.
body_html: |
  HTML template here, or null for WhatsApp.
notes:
  - Any implementation or product caveat.
```

For email + WhatsApp together, return two separate blocks.

## Review Checklist

Before finalizing, check:
- Is the message accurate for the product state?
- Does it avoid implying approval when rejection is possible?
- Is the CTA clear?
- Are sensitive values excluded from WhatsApp?
- Are variables consistently named?
- Is the email subject clear?
- Is the WhatsApp version short enough?
- Does the template name use snake_case?
- Are body_text and body_html aligned?
