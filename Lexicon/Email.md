# Headers

[FlowChart](EmailFlochart.pdf)

### Core delivery

| **Header**     | **Description**                                   | **Notes**                                                                                                      |
| -------------- | ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `From:`        | Declared sender name and address ( header sender) | Easily spoofed; cosmetic only                                                                                  |
| `To:`          | Recipient(s)                                      | Actual envelope recipient                                                                                      |
| `Subject:`     | Email title                                       | Often mimicked in phishing                                                                                     |
| `Date:`        | When the email was composed                       | May differ from server timestamp                                                                               |
| `Message-ID:`  | Unique email ID from sender’s domain              | Often tells what mailer was used (@mail.domain.com)                                                            |
| `Return-Path:` | Where bounces go (envelope sender)                | **May differ from From:**, used for bounce tracking                                                            |
| `Reply-To:`    | Alternate reply address                           | Red flag if it points to a different domain. Not always true, but when it differs it always need investigation |

---
### **Routing Headers**

| **Header**          | **Description**                            | **Notes**                              |
| ------------------- | ------------------------------------------ | -------------------------------------- |
| `Received:`         | List of relays the message passed through  | **Read bottom to top to trace source** |
| `X-Originating-IP:` | (Optional) Client’s IP address             | Useful to locate sender (if present)   |
| `Delivered-To:`     | Final recipient after any alias/forwarding | Useful in multi-account delivery       |
| `X-Forwarded-For:`  | Who originally sent the message (rare)     | Can help detect internal forwarding    |

#### **From Sender to Inbox**

#### **1.** **Composition**
- A user  **writes** an email using a mail client (like Outlook, Thunderbird, webmail, or even a script).
- This client connects to their **outgoing mail server** (`SMTP server`) to send the message.

#### **2.Sending - SMTP = Simple Mail Transfer Protocol**
Sender’s SMTP server **relays** the email by connecting to the **recipient’s MX (Mail Exchange) server**.
Connection usually via **port 25** (465 for SMTPS -> SMTP over SSL) or 587 /w auth, and involves:
- MAIL `FROM:<sender>`
- RCPT `TO:<recipient> `
- DATA (headers + body)

 >**The SMTP protocol does **not** guarantee identity. That’s why **spoofing is easy** without auth.
 >RFC 5321 -> Simple text, kinda like http ?
 >Only sending and relaying, delivery and reading is **NOT** SMTP

#### **3.** **Receiving (MX Lookup & Delivery)**
The sending server uses DNS to **look up MX records** for the recipient’s domain.
- e.g., for user@example.com, it queries: dig MX example.com

It finds the mail server (e.g., mx1.mailhost.com) and delivers the message there. Sometimes there are several, and the highest priority is chosen. 

#### **4. Storage & Retrieval**
The receiving mail server stores the message in the recipient’s **mailbox**.
When the user opens their mail app, they use:
- **IMAP** (port 143 unsecured, 993, default) → to sync/view ( )
- **POP3** (port 110 unsecured, 995) → to download/delete ( AKA *Manual* service, downloads message and deletes from server)
  
These protocols are distinct from SMTP and are used **after** delivery.

#### **Intermediate Relays**

Sometimes, mail goes through **spam filters or security relays** (e.g., Proofpoint, Microsoft 365, Google).  
Each one **adds its own headers** (like Received:, X-*, Authentication-Results:).

---
### **Authentication Headers**
#### 1. `SPF` = Sender Policy Framework
Domain owner, needs to publish a DNS record:
```bash
example.com. TXT "v=spf1 ip4:192.0.2.12 include:_spf.google.com -all"
```
-> links an IP to a specific SMPT
When it arrives to a MTA - **mail transfer agent** - the latter checks the SMTP envelope MAIL FROM and queries the SPF record and check if the ip matches.

> **OUTPUT**
> Pass: IP ok
> Fail: IP not listed
> Softfail: the ip is "likely" not authorized, but not explicitly forbidden. This makes no sense :(
> Neutral:  the domain policy is ambiguous or unstated
> None: No SPF

**SPF is tied to the SMTP Envelope -> return-path**
> `From:` Can be spoofed, but not the return-path

#### 2. DKIM = DomainKeys Identified Mail
The **sending mail server** signs the email headers + body hash with a **private key**
Adds a new header:
```bash
DKIM-Signature: v=1; a=rsa-sha256; d=example.com; s=default; bh=...; b=...
```
`h=` => what header are part of the signature !
So any added header by the MTA will not break signature

The MTA retrieves the matching public key from the DNS:
```bash
default._domainkey.example.com. TXT "v=DKIM1; p=CRYP_KEY..."
```

If signature verifies -> `DKIM: pass`
> Authenticates the domain in the header.
> Confirm the message ( Body + Header) wasn't tempered within transit.

#### 3. DMARC = Domain-based Message Authentication, Reporting, and Conformance
Domain owner pushed a new DNS TXT record:
```bash
_dmarc.example.com. TXT "v=DMARC1; p=reject; rua=mailto:dmarc-reports@example.com"
```
Glues all auth together.
For instance: "Only accept mail if SPF, DKIM pass and `From:` domain matches, otherwise reject, quarantine or just report"

> Policies:
> p=none -> ... monitor/ dont care
> p=quarantine -> junk
> p=reject -> bounce back to return-path

Protects the visible sender! DMARC should be enforced...

#### 4. ARC = ## **Authenticated Received Chain**
So unfortunately DMARC breaks forwarding and mailing lists where:
- Original sender signed it -> DKIM/SPF pass
- But the forwarder modifies or resends -> SPF fails, DKIM fails also.
  > Though i am unsure of this, the forwarded, could be just forwarding a whole body of the sender body, effectively creating a new envelope ?
  > Ok -> There are more than one way to forward message:
  >  - Manual ( click forward ) -> client logic choses what to copy, usually prepends the message with a `FWD` , often copies the subject and `FROM`, but creates a new enveloppe, new SPF, new DKIM.
  >  - Automatic -> Aliases, or MTA forwarding to another mail because the email is just some sort of general email, and rules may distribute the mail to the real clients.
  > 	 - The logic is *transparence* so the SPF and DKIM ( especially DKIM) WILL break, because the header is tempered !

ARC allows intermediary relays ( forwarders, google, outlook etc) to "vouch" for the original auth status..
- Adds a chain of **ARC-Seal**, **ARC-Message-Signature**, and **ARC-Authentication-Results**

| **Header**                    | **Description**                                    | **Notes**                             |
| ----------------------------- | -------------------------------------------------- | ------------------------------------- |
| `Authentication-Results:`     | Shows SPF, DKIM, DMARC results                     | main check for legitimacy             |
| `Received-SPF:`               | Result of SPF check                                | pass, fail, softfail, neutral         |
| `DKIM-Signature:`             | Cryptographic signature from sender                | often includes d=domain.com           |
| `ARC-Authentication-Results:` | Shows result from previous forwarder (ARC chain)   | used in Gmail, Outlook chains, icloud |
| `DMARC-Filter:`               | DMARC enforcement or override (e.g. in MS Outlook) | may show policy like p=reject         |




---
### **Anti-Spam & Analysis Headers**

| **Header**                          | **Description**                        | **Notes**                           |
| ----------------------------------- | -------------------------------------- | ----------------------------------- |
| `X-Spam-Status:`                    | SpamAssassin or similar results        | YES / NO, score shown               |
| `X-Spam-Score:`                     | Numerical spam score                   | Higher = more suspicious            |
| `X-Spam-Flag:`                      | YES if marked as spam                  |                                     |
| `X-Mailer:`                         | Sending app (e.g., Outlook, PHPMailer) | Can expose automation kits          |
| `X-MS-Exchange-Organization-*`      | Internal Microsoft routing headers     | Often contain spam filter verdicts  |
| `X-Proofpoint-*`, `X-Google-DKIM-*` | Headers from filtering services        | Used by large corps (e.g. banks)    |
| `Authentication-Results:`           | Final verdict of DKIM + SPF and ARC    | A single header that summarized all |

---

### **Suspicious Signs and Patterns**

- From: != Return-Path: ≠ DKIM d= domain
- From: != reply-to
- SPF: softfail or fail
- DKIM: fail or missing entirely
- Message-ID from public mailer (e.g. mail-ot1-f65.google.com) but claiming to be from a bank
- Received: from known webmail hosts (e.g., Gmail, Korea University, etc.)
- Reply-To: redirecting to a different domain => Favorite one
- Abnormally generic X-Mailer: (e.g. PHPMailer, python-requests)
- Header Mismatch: The final Received: header (the one at the top) claims to be from a server associated with the From: domain, but the IP address in that same header is from a completely different network (e.g., a data center, a public VPN, a foreign country).
* **Missing or Forged Message-ID**: A legitimate message will always have a valid Message-ID:. An email without one, or with a very generic/unusual one, is often a sign of a bulk mailer or an amateur phishing kit.
* IP Address from a Blocklist: The **X-Originating-IP**: or the IP in a Received: header is on a known public IP blocklist (services like abuseipdb.com or spamhaus.org).
* Unusual Reply-To: Header: it's worth noting cases where the `Reply-To:` is something like `email protected`, designed to hide the real domain in a way that bypasses simple domain checks.
