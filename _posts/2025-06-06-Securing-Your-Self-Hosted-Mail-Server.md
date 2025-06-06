---
layout: post
title: Securing your self-hosted mail Server with SPF, DKIM, DMARC, and MTA-STS
description: Learn how to secure your self-hosted email server with SPF, DKIM, DMARC, and MTA-STS to prevent spoofing, improve deliverability, and protect your domain reputation.
date: 2025-06-06 08:00:00 +0200
tags: [Security, E-Mail]
tags_color: '#44430c'
image: '/images/2025-06-06-Securing-Your-Self-Hosted-Mail-Server/Preview.png'
toc: true
featured: true
---

Running your own email server can be a rewarding experience. You stay in control of your data, reduce your dependence on big tech platforms, and have the freedom to customize everything to fit your needs. But with that control comes responsibility. If your server isn’t properly secured, it can become a target for abuse, or worse, a source of spam or phishing.

Email remains one of the most exploited communication tools today. Attackers often spoof legitimate domains to send fake invoices, phishing links, or impersonation attempts. Without the right protections in place, your domain could be misused, and your legitimate emails may end up in the recipient’s spam folder.

That’s why modern email security protocols are essential. In this article, we’ll walk through four key building blocks that help protect your domain and improve your deliverability:

* SPF: Specifies which servers are allowed to send mail for your domain.
* DKIM: Adds a digital signature to your outgoing mail to verify authenticity.
* DMARC: Tells recipient servers what to do if SPF and DKIM checks fail.
* MTA-STS: Enforces encryption for email delivery between servers.

When implemented together, these standards help establish a strong email security posture — one that protects both your users and your domain’s reputation.

## **SPF: Specify who can send on behalf of your domain**

SPF (Sender Policy Framework) is a DNS-based email authentication protocol that tells other mail servers which IP addresses or systems are allowed to send mail for your domain. It’s your first line of defense against sender spoofing.

When a receiving server gets an email claiming to come from your domain, it checks your domain’s SPF record to verify whether the sending server is authorized. If the IP doesn’t match, the mail can be marked as suspicious or outright rejected.

### How it works

You publish an SPF record as a TXT record in your domain’s DNS. The record lists approved mail servers, like your own server’s IP, your provider’s infrastructure, or services like Mailgun or Google Workspace.

Example SPF record:

```text
example.com IN TXT "v=spf1 mx -all"
```

Let’s break it down:

* **v=spf1** : This declares the version of SPF being used. spf1 is the current and only version in use.
* **mx** : Any server listed in your domain’s MX records is authorized to send email on behalf of your domain.
* **-all** : This is a hard fail. It tells receiving mail servers: Reject any emails that come from sources not listed above.

{: .info }
Tip: Always double-check your SPF syntax. Only one SPF record is allowed per domain, and overly permissive settings (like ~all or +all) can weaken your protection.

### What SPF protects against

SPF helps prevent unauthorized servers from spoofing your domain in the MAIL FROM (envelope sender). It doesn't protect the “From” header that users see in their inbox, that’s where DKIM and DMARC come in.

### Setting it up with Mailcow

If you're running Mailcow, you can configure SPF by:

* Ensuring your public-facing mail server IP is added to your SPF record or "mx".
* Adding necessary includes if you use SMTP relays.
* Publishing the record via your domain's DNS settings.

## **DKIM: Sign your outgoing mail with a cryptographic signature**

DKIM (DomainKeys Identified Mail) adds a digital signature to each outgoing message. It allows receiving mail servers to verify that the email was actually sent by your domain, and that it wasn’t modified in transit.

This helps prevent tampering, ensures message integrity, and builds trust with other mail servers.

### How it works

When your mail server sends an email, it generates a unique cryptographic signature based on the message’s headers and body. That signature is added to the email as a hidden header called DKIM-Signature.

On the receiving side, the server uses your domain’s public DKIM key (published as a TXT record in DNS) to validate the signature. If the signature checks out, the message is considered authentic.

Example DKIM DNS record (TXT):

```text
dkim._domainkey.example.com IN TXT "v=DKIM1; k=rsa; p=MIGfMA0G...QAB"
```

Let’s break it down:

* **dkim** : Is the selector, which lets you use multiple DKIM keys if needed.
* **_domainkey** : Is part of the required DKIM DNS structure.
* **example.com** : Is your domain name.
* **v=DKIM1** : Indicates the DKIM version.
* **k=rsa** : Specifies the encryption algorithm.
* **p=..** : This is your public key. Receiving mail servers use this to verify the digital signature in your emails.

### What DKIM protects against

DKIM helps ensure:

* The email really came from your domain.
* The content wasn’t modified after sending.
* Spoofed emails without a valid signature can be flagged or rejected (especially when combined with DMARC).

Unlike SPF, DKIM protects the visible “From” address in the email header — the part users actually see.

### Setting it up with Mailcow

Mailcow makes DKIM easy:

* It automatically generates DKIM keys per domain.
* You’ll find them in the Mailcow admin panel under Configuration > Options > ARC/DKIM Keys.
* Just publish the DNS TXT record it shows for your domain.

Once set, you can test DKIM by sending an email to a Gmail address and viewing the “Original message” (check for DKIM=pass).

## **DMARC: Define a policy and gain visibility**

DMARC (Domain-based Message Authentication, Reporting & Conformance) is the protocol that ties SPF and DKIM together and tells receiving servers what to do when a message fails authentication checks.

Without DMARC, even if you have SPF and DKIM, a forged email might still appear trustworthy to some recipients. DMARC provides that missing link by letting domain owners define clear rules and monitor abuse attempts.

### How it works

DMARC builds on SPF and DKIM by checking:

* Does the email pass SPF or DKIM?
* Does the domain in the "From" header align with the authenticated domain?

If checks fail, DMARC tells the receiving server what to do (none, quarantine, or reject).

Example DKIM DNS record (TXT):

```text
_dmarc.example.com IN TXT "v=DMARC1; p=reject; rua=mailto:dmarc-reports@example.com; adkim=s; aspf=s"
```

Let’s break it down:

* **_dmarc** : Is part of the required DMARC DNS structure.
* **v=DMARC1** :  DMARC version.
* **p=reject** : Policy if checks fail (none | quarantine | reject).
* **rua=mailto:...** : Where to send aggregate reports.
* **aspf=s** : Strict SPF alignment (header From must exactly match).
* **adkim=s** : Strict DKIM alignment.

{: .info }
Tip: Use p=none when starting out, you’ll get reports but no mail will be blocked. Once you’re confident, switch to quarantine or reject.

### What are DMARC reports?

DMARC lets you receive aggregate reports (XML files) from other mail providers showing:

* Who is sending mail using your domain
* Whether SPF and DKIM passed
* IPs attempting to spoof your domain

### Setting it up with Mailcow

Mailcow handles DKIM and SPF, but DMARC is configured at the DNS level:

* Decide on your policy (none, quarantine, or reject)
* Create a DNS TXT record at _dmarc.yourdomain.com
* Point rua to an email inbox where you can receive and analyze reports

Once active, monitor your reports for at least a week before enforcing stricter policies.

## **MTA-STS: Enforce secure mail transport between servers**

MTA-STS (Mail Transfer Agent Strict Transport Security) protects email in transit between mail servers. It ensures that emails sent to your domain are only delivered over encrypted TLS connections, and only to servers you trust.

Without MTA-STS, attackers could intercept or redirect mail using DNS spoofing or downgrade attacks (e.g., forcing unencrypted delivery). MTA-STS prevents this by publishing a strict transport policy that sending servers can verify.

### How it works

MTA-STS works in two parts:

1. A DNS record (a TXT record) tells other mail servers where to find your policy.
2. An HTTPS-hosted policy file describes which mail servers are valid and whether TLS is required.

If the sending server supports MTA-STS, it will:

* Fetch the policy via HTTPS
* Check that the destination mail server’s certificate is valid
* Only deliver mail if the connection is secure and matches your policy

Example MTA-STS DNS record (TXT):

```text
_mta-sts.example.com IN TXT "v=STSv1; id=20250526T000000;"
```

Let’s break it down:

* **_mta-sts** : Is part of the required MTA-STS DNS structure.
* **v=STSv1** :  The current version.
* **id=** : A unique ID; update this whenever your policy file changes. Can be anything you want.

### Example policy file

The policy file must be available at https://mta-sts.example.com/.well-known/mta-sts.txt

The contents would be something like this:

```text
version: STSv1
mode: enforce
mx: mail.example.com
max_age: 604800
```

* mode: enforce — Only accept secure delivery.
* mx: — List your valid MX hostnames.
* max_age: — How long (in seconds) other servers should cache the policy.

You’ll need to host this file on a web server with a valid TLS certificate.

### Create a docker container to serve the file

I am running traefik as my proxy, this alows me to create a docker container that uses labels for easy hosting for the MTA-STS policy file.

1. Create a seperate directory for the docker container (e.g. /home/docker/mta-sts-container) and create the docker-compose.yml file:

```yaml
version: "3.8"

services:
  mta-sts-server:
    image: nginx:alpine # NGINX running on Alpine image
    volumes:
      - ./mta-sts:/usr/share/nginx/html/.well-known:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.mta-sts.rule=Host(`mta-sts.example.com`)"
      - "traefik.http.routers.mta-sts.entrypoints=https"
      - "traefik.http.routers.mta-sts.tls=true"
      - "traefik.http.services.mta-sts.loadbalancer.server.port=80"
    networks:
      - proxy

networks:
  proxy:
    external: true  # Same network as the traefik container
```

2. In the /home/docker/mta-sts-container directory create a directory named mta-sts and create the mta-sts.txt file with contents:

```text
version: STSv1
mode: enforce
mx: mail.example.com
max_age: 604800
```

3. Execut "sudo docker compose up -d" in the /home/docker/mta-sts-container directory and the file should be accessible by browser at "https://mta-sts.example.com/.well-known/mta-sts.txt
"

### MTA-STS Reporting

If you’re using MTA-STS, you can also enable TLS Reporting (TLS-RPT). This lets other mail servers send reports about delivery issues (e.g., failed TLS handshakes).

Just add this TXT DNS record:

```text
_smtp._tls.example.com IN TXT "v=TLSRPTv1; rua=mailto:tls-reports@example.com"
```

Let’s break it down:

* **_smtp._tls** : Is part of the required MTA-STS TLS Reporting DNS structure.
* **v=TLSRPTv1** :  The current version.
* **rua=mailto:** : Where to send TLS MTA-STS reports.

***

Securing your own e-mail server isn’t just about getting messages delivered, it’s about protecting your domain, your users, and your reputation. By implementing SPF, DKIM, DMARC, and MTA-STS, you build a modern, standards-based defense against spoofing, phishing, and man-in-the-middle attacks.

While these protocols might seem complex at first, they’re essential for anyone serious about running email responsibly. Take the time to configure them correctly, monitor the reports, and update your policies as needed, your future self (and your users) will thank you.