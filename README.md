# Google Review Reply Agent

A webhook for an HVAC business that drafts a reply to each Google review. A negative review (1 to 3 stars) gets an apology and an invitation to contact the team, written by OpenAI. A positive review gets a fixed thank-you message. Every review and its reply is saved to Supabase, and the reply is returned to the caller so it can be posted on Google.

## Architecture

```
Webhook (POST /google-review)
   → Validate Input (review text not empty, rating 1 to 5)
        ├── Invalid → Respond Invalid (400)
        └── Valid → Check Rating
                     ├── 1 to 3 stars → Negative Reply (OpenAI)
                     │                    → Normalize Negative Reply
                     └── 4 to 5 stars → Positive Reply (fixed text)
                     → Merge Replies
                     → Save Review (Supabase: review_replies)
                     → Respond Success (returns the reply)
```

## Features

* Checks that the review text is not empty and the rating is between 1 and 5
* Accepts the rating as a number or as text (`"4"`)
* Negative reviews get a short, empathetic reply written by OpenAI. It apologizes, names the problem, invites the customer to call, and makes no promises about refunds or compensation
* Positive reviews get a fixed thank-you message, so the AI is not used for easy cases
* Replies are written in the same language as the review
* Every review, its rating, sentiment, and reply are saved to Supabase
* The reply is returned in the response, so the caller can post it to Google

## Tech Stack

* n8n 2.x (workflow automation, webhooks)
* OpenAI `gpt-4o-mini` (negative review replies)
* Supabase (review_replies table)

## Setup

1. Create a Supabase table named `review_replies` with these columns:

```
id | reviewer_name | rating | review_text | reply | sentiment | created_at
```

2. Import `google_review_reply_agent.json` into n8n.
3. Add credentials: OpenAI and Supabase. Assign them to the matching nodes.
4. Activate the workflow.
5. Send a test review.

## Sample Requests

Negative (AI reply):

```
POST https://YOUR_N8N_DOMAIN/webhook/google-review
Content-Type: application/json

{
  "reviewer_name": "Ali Raza",
  "rating": 2,
  "review_text": "The technician arrived two hours late and the AC is still not cooling."
}
```

Positive (fixed reply):

```
{
  "reviewer_name": "Sara Khan",
  "rating": 5,
  "review_text": "Quick and friendly service. The house is cool again."
}
```

Expected result: a `review_replies` row and a response like:

```
{ "status": "ok", "reply": "..." }
```

Invalid input returns HTTP 400 with `{"status": "invalid", ...}`.

## Known Limits

* This workflow does not post the reply to Google. It returns the reply and saves it. Posting to Google Business Profile needs a separate step (for example a manual copy-paste or the Business Profile API with Google OAuth).
* The reply is generated automatically. Read the negative replies before they are posted, especially when the review names a specific person or a serious problem.
* Positive replies use one fixed message. If you want more variety, add a few templates and pick one at random.

## Project Notes

* Built as part of my AI Automation portfolio.
* The AI is used only for negative reviews, where a personal tone matters. Positive reviews are handled in code.
* Only the OpenAI reply node uses `gpt-4o-mini`. Change its model in that node if you want another model.
