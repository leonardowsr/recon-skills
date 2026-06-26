---
name: proton-mail-verifier
description: Login to Proton Mail inbox via Playwright, search for verification/confirmation emails, extract OTP codes and verification links. Essential for automating account creation flows that require email confirmation.
version: 1.0.0
author: agentiko
license: MIT
platforms: [linux, macos]
compatibility: Requires python3, playwright (chromium), and a valid Proton Mail account (free tier works).
metadata:
  hermes:
    tags: [recon, email, proton-mail, verification, account-creation, automation]
    category: recon
    related_skills:
      - js-secrets-extraction
      - firebase-supabase-attack
      - hunt-api-misconfig
---

# Proton Mail Verifier

Login to a Proton Mail inbox via Playwright headless browser, search for specific emails (verification, confirmation, OTP), and extract links/codes. Essential for automated account creation flows that send confirmation emails — no more manual inbox checking during recon.

## When to Use

- Target sends email verification/confirmation after account registration.
- You need to extract a verification link or OTP code from an email automatically.
- Building an automated account creation → email verification → authenticated API access chain.
- Any recon flow that hits an "email_not_verified" wall and needs to complete the verification step.

## Prerequisites

- `python3` with `playwright` installed (`pip install playwright && playwright install chromium`)
- A valid Proton Mail account (free tier on proton.me works fine).
- The email address and password for that account.

## How to Run

### Quick Start — Extract verification links

```bash
python3 proton_verify.py \
  --email "user@proton.me" \
  --password "your_password" \
  --sender "target-domain.com" \
  --wait 15
```

Returns:
```
Found 2 emails from target-domain.com
Email 1: "Confirm your email"
  🔗 https://target.com/verify?token=abc123...
  🔗 https://target.com/confirm?code=xyz789...
```

### Search by keyword in subject

```bash
python3 proton_verify.py \
  --email "user@proton.me" \
  --password "pass" \
  --keyword "verification,confirm,verify,OTP" \
  --wait 10
```

### Extract OTP codes (6-digit numbers)

```bash
python3 proton_verify.py \
  --email "user@proton.me" \
  --password "pass" \
  --sender "auth@target.com" \
  --extract-otp
```

### List all recent emails

```bash
python3 proton_verify.py \
  --email "user@proton.me" \
  --password "pass" \
  --list
```

## Python Script — `proton_verify.py`

```python
#!/usr/bin/env python3
"""
Proton Mail Verifier — Login, search inbox, extract links/OTP codes.
Uses Playwright headless browser to automate Proton Mail login.
"""

import asyncio, argparse, re, sys, time, json, os
from playwright.async_api import async_playwright

PROTON_LOGIN = "https://mail.proton.me/u/0/login"

async def login(page, email, password):
    """Login to Proton Mail. Returns True on success."""
    await page.goto(PROTON_LOGIN, wait_until="domcontentloaded", timeout=30000)
    await page.wait_for_timeout(2000)
    
    try:
        await page.fill('#username', email)
        await page.fill('#password', password)
        await page.click('button[type="submit"]')
    except Exception as e:
        print(f"  ❌ Form fill error: {e}")
        return False
    
    try:
        await page.wait_for_url("**/inbox**", timeout=30000)
        await page.wait_for_timeout(4000)
        return True
    except:
        text = await page.evaluate("document.body.innerText")
        if "credenciais" in text.lower() or "invalid" in text.lower():
            print(f"  ❌ Invalid credentials")
        elif "captcha" in text.lower():
            print(f"  ❌ CAPTCHA detected — need manual solve")
        else:
            print(f"  ❌ Login failed: {text[:200]}")
        return False


async def get_inbox_emails(page):
    """Extract email subjects and metadata from inbox."""
    return await page.evaluate("""() => {
        const items = document.querySelectorAll('[data-testid*="conversation"], [data-shortcut-target*="conversation"], [class*="item-container"]');
        const emails = [];
        items.forEach((el, i) => {
            const text = el.textContent?.trim().replace(/\\s+/g, ' ').slice(0, 300);
            if (text && text.length > 5) {
                emails.push({index: i, element: el, text: text});
            }
        });
        return emails;
    }""")


async def open_email(page, email_index):
    """Click an email in the inbox and wait for it to load."""
    emails = await get_inbox_emails(page)
    if email_index >= len(emails):
        return None
    
    convs = await page.query_selector_all('[data-testid*="conversation"]')
    if email_index < len(convs):
        await convs[email_index].click()
        await page.wait_for_timeout(5000)
    
    # Wait for content iframe (Proton uses sandboxed iframes for email content)
    try:
        await page.wait_for_selector('iframe', timeout=8000)
    except:
        pass
    
    return await page.content()


def extract_links(html, domain_filter=None):
    """Extract all absolute URLs from HTML, optionally filtered by domain."""
    urls = re.findall(r'https?://[^\s"\'<>&]+', html)
    if domain_filter:
        urls = [u for u in urls if domain_filter.lower() in u.lower()]
    return list(set(urls))  # deduplicate


def extract_otp_codes(text):
    """Extract 4-8 digit OTP/verification codes from text."""
    # Common OTP patterns
    patterns = [
        r'\b(\d{4,8})\b',                    # Standalone 4-8 digits
        r'code[:\s]*(\d{4,8})',              # "code: 123456"
        r'código[:\s]*(\d{4,8})',            # "código: 123456"
        r'OTP[:\s]*(\d{4,8})',               # "OTP: 123456"
        r'verification code[:\s]*(\d{4,8})', # "verification code: 123456"
        r'PIN[:\s]*(\d{4,8})',               # "PIN: 123456"
    ]
    codes = set()
    for pattern in patterns:
        matches = re.findall(pattern, text, re.I)
        codes.update(matches)
    return list(codes)


async def main():
    parser = argparse.ArgumentParser(description="Proton Mail Verifier")
    parser.add_argument("--email", required=True, help="Proton Mail email address")
    parser.add_argument("--password", required=True, help="Proton Mail password")
    parser.add_argument("--sender", help="Filter emails from this domain or sender")
    parser.add_argument("--keyword", help="Comma-separated keywords to filter subjects")
    parser.add_argument("--list", action="store_true", help="Just list recent emails")
    parser.add_argument("--extract-otp", action="store_true", help="Extract OTP codes from emails")
    parser.add_argument("--wait", type=int, default=10, help="Seconds to wait after login (default: 10)")
    parser.add_argument("--json", action="store_true", help="Output as JSON")
    args = parser.parse_args()
    
    keywords = [k.strip().lower() for k in args.keyword.split(",")] if args.keyword else []
    
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True, args=['--no-sandbox'])
        context = await browser.new_context(
            user_agent="Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36",
            locale="pt-BR",
            viewport={"width": 1280, "height": 720}
        )
        page = await context.new_page()
        
        if not await login(page, args.email, args.password):
            await browser.close()
            sys.exit(1)
        
        print(f"✅ Logged in as {args.email}")
        
        # Optional: wait for new emails to arrive
        if args.wait > 0 and (args.sender or keywords):
            print(f"   Waiting {args.wait}s for new emails...")
            await page.wait_for_timeout(args.wait * 1000)
        
        emails = await get_inbox_emails(page)
        print(f"\n📬 {len(emails)} emails in inbox")
        
        results = []
        for i, email in enumerate(emails):
            subject = email['text']
            
            # Filter by sender
            if args.sender and args.sender.lower() not in subject.lower():
                continue
            
            # Filter by keywords
            if keywords and not any(kw in subject.lower() for kw in keywords):
                continue
            
            if args.list:
                results.append({"index": i, "subject": subject})
                continue
            
            print(f"\n📧 [{i}] {subject[:150]}")
            
            # Open the email
            html = await open_email(page, i)
            if not html:
                continue
            
            # Extract links
            links = extract_links(html, args.sender if args.sender else None)
            for link in links:
                if any(x in link.lower() for x in ['verification', 'verify', 'confirm', 'self-service', 'login', 'auth', 'token']):
                    print(f"  🔗 {link}")
                    results.append({"type": "verification_link", "email_index": i, "subject": subject[:100], "url": link})
                elif args.sender and args.sender.lower() in link.lower():
                    print(f"  🔗 {link}")
                    results.append({"type": "link", "email_index": i, "subject": subject[:100], "url": link})
            
            # Extract OTP codes
            if args.extract_otp:
                codes = extract_otp_codes(html)
                for code in codes:
                    print(f"  🔢 OTP: {code}")
                    results.append({"type": "otp", "email_index": i, "subject": subject[:100], "code": code})
            
            # Also check the page-level HTML (Proton might render content differently)
            page_html = await page.content()
            if args.extract_otp:
                codes = extract_otp_codes(page_html)
                for code in codes:
                    if not any(r.get('code') == code for r in results):
                        print(f"  🔢 OTP (page): {code}")
                        results.append({"type": "otp", "email_index": i, "code": code})
            
            # Go back to inbox
            await page.goto("https://mail.proton.me/u/0/inbox", wait_until="domcontentloaded", timeout=15000)
            await page.wait_for_timeout(2000)
        
        if args.json:
            print(json.dumps(results, indent=2, ensure_ascii=False))
        
        await browser.close()
        
        if not args.list and not results:
            print("\n⚠️ No matching emails found. Try --list to see all emails.")


if __name__ == "__main__":
    asyncio.run(main())
```

## Integration Pattern — Full Account Creation Chain

```python
# Example: Create account → verify email → get JWT → access API
import asyncio, requests, re, json

TARGET = "https://target.com"
EMAIL = "user@proton.me"
PASSWORD = "your_password"

# Step 1: Create account on target
r = requests.post(f"{TARGET}/api/signup", json={
    "email": EMAIL,
    "password": PASSWORD,
    "name": "Test"
})
print(f"Account created: {r.status_code}")

# Step 2: Wait for verification email and extract link via proton-verify skill
proc = await asyncio.create_subprocess_exec(
    "python3", "proton_verify.py",
    "--email", EMAIL,
    "--password", PASSWORD,
    "--sender", TARGET.replace("https://", ""),
    "--wait", "15",
    "--json",
    stdout=asyncio.subprocess.PIPE
)
stdout, _ = await proc.communicate()
results = json.loads(stdout)

# Step 3: Follow verification link
if results:
    verify_url = results[0]['url']
    r = requests.get(verify_url)
    print(f"Verified: {r.status_code}")

# Step 4: Login and get token
r = requests.post(f"{TARGET}/api/login", json={
    "email": EMAIL,
    "password": PASSWORD
})
token = r.json().get('token')
print(f"JWT: {token[:50]}...")
```

## Pitfalls

- **Proton Mail loading is slow**: The SPA takes several seconds to fully render. Increase `--wait` if emails don't appear immediately.
- **CAPTCHA**: Proton may require CAPTCHA if the IP is suspicious. Switch to a residential proxy or different exit node.
- **New emails may not appear instantly**: Some email providers batch-send emails. Use `--wait` to give time for delivery.
- **Two inbox views**: Proton has user 0 and user 1 inboxes (`/u/0/inbox` vs `/u/1/inbox`). The script uses `/u/0` by default. Adjust if needed.
- **Email content in iframes**: Proton renders email bodies in sandboxed iframes. The script attempts to access the iframe content but may fall back to page-level link extraction.
- **Proton rate limits**: Rapid repeated logins may trigger security measures. Space login attempts at least 30s apart.

## Verification

- Must successfully log into Proton Mail and display inbox count.
- Must find and open an email from the specified sender.
- Must extract at least one verification URL or OTP code from the email body.
- The extracted link must point to the target domain (not Proton or tracking domains).
