# Email OSINT: How to Build an Investigation from a Single Address

> **In brief.** An email address is one of the most persistent identifiers in a digital footprint: it is rarely changed, it is used to register almost everywhere, and it ends up in data breaches, git commits, document metadata, and DNS records. But the value comes not from any single tool, it comes from a methodology: **address analysis → validation → provider ecosystem → linked services → breaches → public mentions → correlation and confidence assessment**. This article covers the techniques, their limitations, the risk of exposing yourself, and the legal framework.

---

## Why Email Is a Strong Starting Point

A person's name is not unique, a username is easy to change, and a phone number is changed rarely but is tied to a carrier and a country. Email occupies a special place:

- **It is a login.** The address serves as the primary identifier at registration, so it can be used to find accounts across dozens of services.
- **It lives for years.** People carry the same address across services, jobs, and devices.
- **It leaves traces in many different places:** database leaks, WHOIS, git commits, PGP keys, PDF/DOCX metadata, forums, résumés.
- **Corporate addresses have a structure** (`name.surname@company.com`) that makes it possible to reconstruct colleagues' addresses and the naming scheme.

An important caveat: **an email address is not an identity.** It may be a functional address (`info@`), shared by several people, reused after a change of owner, or registered to a stranger by mistake. The entire methodology below is built around this: every finding is a hypothesis until it is confirmed by an independent source.

---

## Legal and Ethical Framework

Email OSINT works with personal data, so it is worth defining the boundaries before the first query.

- **Basis and purpose.** An investigation must have a lawful purpose: vetting a counterparty, incident response, journalistic or academic research, an authorized pentest, or an audit of your own attack surface. "Curiosity" is not a basis.
- **Passive vs. active interaction.** Searching open sources is one thing. Sending requests to services (registration checks, account recovery) is already interaction with someone else's infrastructure and possibly someone else's account. Service terms and the laws of many countries (including those on unauthorized access) may treat such actions strictly.
- **Breaches are data, not keys.** Passwords found in breaches must never be used to log in to accounts, under any circumstances: that is unauthorized access. A breach is used for attribution and exposure assessment.
- **Data minimization.** Collect what the task requires, keep it for the minimum necessary period, and protect the results.
- **Authorization.** Active checks during a pentest or red team engagement require a scope agreed in writing.

Everything marked **active** below should be applied only with such authorization, or to your own resources.

---

## The Investigation Workflow

```
                      Email address
                            │
   ┌────────────────────────┼─────────────────────────┐
   ▼                        ▼                         ▼
0. Address and         1. Validation            2. Provider
   domain analysis        (SMTP)                   ecosystem
                                                   (Google, etc.)
   └────────────────────────┼─────────────────────────┘
                            ▼
              3. Linked services and accounts
                            ▼
                     4. Data breaches
                            ▼
        5. Public mentions (dorking, git, archives)
                            ▼
     6. Correlation → confidence assessment → report
```

Transition points (**pivots**) you will pick up along the way: a name, a username, a phone number, a photo, a numeric account ID, a domain, other email addresses. Each pivot starts a new search cycle.

**Passive and active methods:**

| Method | Type | What happens |
|---|---|---|
| Address structure analysis, domain DNS/WHOIS | Passive | Public reference data |
| Search queries, archives, git, Gravatar | Passive | You query search engines and public resources |
| Breach database lookups (HIBP, IntelX) | Passive | A query to a third-party index |
| SMTP verification | Semi-active | A connection to the domain's mail server |
| Service registration checks | Active | Requests to service endpoints using someone else's address |
| Account recovery | Active | Initiating a procedure on the account |

Start with passive methods and move to active ones only when necessary.

---

## Step 0. Analyzing the Address Itself

Before running any tools, examine the address by eye.

**Structure and type:**

- **Public mailbox provider** (`gmail.com`, `outlook.com`, `proton.me`): identification is possible only through the account and its traces.
- **Corporate or personal domain:** a separate layer of data (RDAP/WHOIS, registration history, DNS, subdomains).
- **Role-based** (`info@`, `support@`, `admin@`): the address belongs to a role, not a person. Investigating a "person" through it is rarely productive.
- **Disposable** (throwaway mail): the service can be identified from the domain, but there is usually no point in looking for an owner.

**Aliases and normalization.** Gmail ignores dots in the local part (`n.ame@gmail.com` and `name@gmail.com` are the same mailbox), supports the `+tag` suffix, and the `googlemail.com` domain. Practical takeaway: if `name+shopname@gmail.com` appears in a breach or in correspondence, the tag shows which service the address was used with, and therefore which service may have been the source of the leak.

**The local part as a source of usernames.** The part before the `@` often matches a username on other platforms. It is the first candidate to check against username databases (see WhatsMyName below).

**Domain analysis** (if it is not a public provider):

- `MX` records reveal the mail provider: Google Workspace, Microsoft 365, Proton, and so on.
- `SPF`, `DKIM`, and `DMARC` describe the mail infrastructure and the maturity of its security configuration.
- WHOIS/RDAP and historical WHOIS can reveal the owner, the registrar, and related domains.
- Certificate Transparency (for example, `crt.sh`) helps find subdomains and related services.
- For corporate mail, identify the pattern (`name.surname@`, `n.surname@`, `surname@`): it lets you form hypotheses about other employees' addresses.

---

## Step 1. SMTP Verification: Does the Mailbox Exist?

**SMTP verification** checks whether a mail server accepts an address without sending an email. In an investigation it serves as an **initial filter**: the address is valid, invalid, or worth further work.

**How it works:**

1. The domain's `MX` record is identified.
2. A connection to the SMTP server is established.
3. The `EHLO`, `MAIL FROM`, and `RCPT TO` commands are sent with the address being checked.
4. The mailbox's existence is inferred from the response code. The session is terminated before the `DATA` command, so no email is sent.

```
S: 220 mx.example.com ESMTP
C: EHLO research.example.net
C: MAIL FROM:<check@research.example.net>
C: RCPT TO:<target@example.com>
S: 250 OK               ← the mailbox accepts mail
S: 550 No such user     ← the mailbox does not exist
C: QUIT
```

**Main response codes:**

| Code | Meaning |
|---|---|
| `250` | Address accepted |
| `550`, `551`, `553` | Mailbox does not exist or is unavailable |
| `450`, `451`, `452` | Temporary error (including greylisting); retry later |
| `421` | Server unavailable or limiting connections |

**Limitations that are often forgotten:**

- **Catch-all domains** accept any address, so `250` does not confirm that a specific mailbox exists.
- **Large providers** deliberately limit how informative their responses are and block IPs with a poor reputation.
- **Outbound port 25** is blocked by many hosting providers and home ISPs.
- **Frequent checks from a single IP** get you onto blocklists and damage the reputation of your infrastructure.
- **"Valid" only means "the mailbox accepts mail."** It does not confirm that a specific person is behind it.

For this reason, third-party services are often used in practice, for example [emailvalidator.io](https://emailvalidator.io/). In addition to SMTP, they check syntax, domain, `MX`, disposable and role-based flags, aliases, plus tags, and `SPF/DKIM/DMARC`. The flip side: you are handing the address you are checking to a third party. In sensitive investigations, that is also a leak of your interest.

**Example of SMTP verification results:**

![SCREENSHOT HERE](https://pbs.twimg.com/media/HK7LbbtWoAAdtXs?format=jpg&name=medium)

---

## Step 2. The Provider Ecosystem: Google and Epieos

For Gmail addresses and Google accounts, there is a separate and highly productive layer of data. **[Epieos](https://epieos.com/)** is an OSINT platform that aggregates publicly available data on emails and phone numbers. In email investigations it is most often used to analyze Google accounts.

**What you can obtain:**

- profile name and photo, if they are public;
- **the numeric Google account identifier** (Google ID / Gaia ID): a stable pivot that does not change when the name or photo changes;
- **Google Maps:** reviews, ratings, photos. These reveal places, habits, and the area where someone lives or works;
- **Google Calendar:** public calendars and events, if the owner has made them open;
- **Google+ archives:** the service was shut down in 2019, but copies may survive in the Wayback Machine;
- linked platforms and external resources.

**Example of Epieos output:**

![SCREENSHOT HERE](https://pbs.twimg.com/media/HK7LlP1WUAAFksa?format=jpg&name=900x900)

**How to read the result:**

- Data is present **only where the user has left it public.** An empty result does not mean the account does not exist.
- The output includes a *Last Update* field: the data may be stale.
- Google regularly changes its endpoints, so tools of this class break from time to time. Before you start work, check that the tool is current and cross-check the result against another source.
- Open-source alternative: **GHunt** (CLI). It needs the cookies of a Google account, so use only a dedicated research account.

**Other sources keyed to an email address:**

- **Gravatar.** An avatar and profile are tied to the MD5 hash (SHA-256 is now also supported) of the normalized address. If the user created a Gravatar, the hash yields a name, photo, and links to accounts, and the request itself does not send the address in cleartext.
- **The Microsoft ecosystem** (Outlook/Live): linked Xbox, Skype, and other service accounts may reveal a name and a username.

---

## Step 3. Linked Services and Accounts

After validation, the next task is to understand **where the address is used as a login.** This is how a map of the digital footprint is built: music and gaming services, shops, social networks, forums, SaaS.

**How such tools work:**

- testing **password recovery mechanisms** (a service responds differently to an existing and a nonexistent address);
- analyzing **public endpoints** for registration and availability checks (`email already taken`);
- recognizing **characteristic response patterns**: text, status code, structure, sometimes response time;
- **correlating** with data from open sources.

**Tools:**

| Tool | What it does | Input |
|---|---|---|
| **[Blackbird](https://github.com/p1ngul1n0/blackbird)** | CLI tool for finding accounts by email and username; can extract metadata | email, username |
| **[Holehe](https://github.com/megadose/holehe)** | Checks whether an address is registered on many services via recovery mechanisms | email |
| **[WhatsMyName](https://github.com/WebBreacher/WhatsMyName)** | A continuously updated database of site signatures; mainly solves **username lookup** | username |

Note the distinction: WhatsMyName is about usernames. Its role in an email investigation arises when a username has been derived from the address (or from discovered accounts) and needs to be checked on other platforms.

**Example of Blackbird output:**

![SCREENSHOT HERE](https://pbs.twimg.com/media/HK7LtyRWYAAh6uE?format=jpg&name=medium)

The most valuable thing here is not the fact of registration itself but the **metadata**: user ID, avatar, registration date, public name. These become new pivots.

**What is important to understand:**

- `FOUND` means the service responded as it would for an existing account, not that the account is active or belongs to the target.
- **False positives and false negatives** are possible: services change their logic, protect against enumeration, and spoof responses.
- Some checks work through the password recovery mechanism, and **some of them may send the owner an email or a notification.** Before running a tool, read its documentation and assess the risk of exposure.
- This is an **active** method: mass enumeration violates the rules of many services and leads to IP blocks.

---

## Step 4. Data Breaches

Checking an address against public breaches is a key step in email OSINT. It shows **where and when the address was exposed, and what data was disclosed as a result.**

**What breaches typically contain:**

- email addresses and usernames;
- hashes and (less often) plaintext passwords;
- account registration dates;
- IP addresses;
- phone numbers, names, addresses, dates of birth, depending on the service.

**Tools:**

| Tool | Purpose |
|---|---|
| **[Have I Been Pwned](https://haveibeenpwned.com/)** | Shows which known breaches contain the address and which categories of data were exposed. Does not show passwords |
| **[Intelligence X](https://intelx.io/)** | A search platform for leaked and indexed data: breaches, paste sites, and other sources |
| **[Leak-Lookup](https://leak-lookup.com/)** | A breach search service for emails and credentials |

**Example of Have I Been Pwned output:**

![SCREENSHOT HERE](https://pbs.twimg.com/media/HK7L4uIXwAEc6rX?format=jpg&name=medium)

**How to extract analytical value from breaches:**

- **Age of the address.** The earliest breach sets a lower bound on when the address has been in use.
- **Interests and profile.** The list of services (games, forums, dating, cryptocurrency) characterizes the person's areas of activity.
- **New pivots.** Usernames, phone numbers, IPs, and names can be pulled from breach records.
- **Timeline.** Registration and breach dates help tie events together.

**Limitations and risks:**

- **Presence in a breach ≠ ownership of the address.** People register with other people's addresses, make typos, and spam bots insert random addresses.
- **Data quality.** Aggregated "combo lists" and repackaged databases often contain junk, duplicates, and fabrications.
- **Do not test found passwords against services.** See section 2.
- For defensive tasks, [Pwned Passwords](https://haveibeenpwned.com/Passwords) with k-anonymity is useful: it lets you check your own password without revealing it.

---

## Step 5. Public Mentions: Google Dorking and Beyond

**Google Dorking** is the use of advanced search operators to find public mentions of an address: on websites, forums, in documents, on paste services, and in indexed files. With well-crafted queries you can find:

- forum posts and comments;
- paste-service publications;
- documents and exposed files;
- public profiles and contact pages;
- résumés, PDFs, DOCX files, and spreadsheets;
- technical logs and configuration files.

**Basic dorks:**

```text
# Exact match
"example@gmail.com"

# Search on specific sites
site:github.com "example@gmail.com"
site:vk.com "example@gmail.com"
site:pastebin.com "example@gmail.com"

# By file type
filetype:pdf "example@gmail.com"
filetype:doc "example@gmail.com"
filetype:xls "example@gmail.com"

# Open directories and listings
intitle:"index of" "example@gmail.com"

# Logs and configurations
"example@gmail.com" "password"
"example@gmail.com" "login"
```

The last group of queries is for exposure assessment: it shows whether credentials have ended up in the open. Any passwords found must not be used (see section 2), and for critical findings it is better to notify the owner.

**How to make it more effective:**

- **Search for spelling variants.** People obfuscate addresses: `name [at] example [dot] com`, `name(at)example.com`.
- **Search the local part separately** (`"username"` together with `site:`): it often matches a username.
- **Use several search engines.** The indexes of Google, Bing, Yandex, DuckDuckGo, and Brave differ. For the Russian-language segment, Yandex often finds what Google does not.
- **Check archives.** The Wayback Machine and caches preserve deleted pages.

**Other places where addresses live:**

- **Git commits.** Commit metadata stores the author's email. It can be seen in the `.patch` version of a commit on GitHub, or via `git log --format='%ae'` in a cloned repository. Some developers use GitHub's `noreply` address, which reduces the value of this source.
- **Document metadata** (`exiftool`): author and last-modified-by fields.
- **PGP keyservers:** keys list addresses and names.
- **WHOIS and its history:** registrant contact data, especially for older domains.

---

## Step 6. Account Recovery: Masked Data

Some services, during password recovery, show partially hidden contact details: the last digits of a phone number, a country code, a masked backup address such as `j***@outlook.com`.

**Why it is useful:**

- to refine the target's country or region;
- to narrow the range of options by matching the mask against phone numbers from breaches or social networks;
- to determine which mail system the target uses (from the backup address's domain);
- to obtain additional pivots for correlation.

**Why it is the riskiest method:**

- **It is active interaction with someone else's account.** Starting a recovery procedure can trigger a **notification to the owner** (an email, a push to their devices). The target learns that someone is interested in their account.
- **The provider logs the request:** IP address, region, browser data. Frequent requests lead to CAPTCHAs and temporary restrictions.
- **Legal risk.** Depending on the jurisdiction and the service's rules, an attempt to initiate recovery of someone else's account may be classified as an attempt at unauthorized access.
- **Providers increasingly limit such hints**, so the method is unstable.

Practical takeaway: apply it only to your own accounts or within an authorized pentest with an agreed scope, and otherwise prefer passive sources that give a similar result (breaches, mentions, Gravatar).

---

## Correlation and Confidence Assessment

No single technique is effective on its own. Value emerges where sources intersect: SMTP confirms the address, breaches show historical exposure, linked accounts provide a map of services, and dorking, Google data, and recovery hints add context.

**Pivot map:**

| What was found | Where to develop it |
|---|---|
| Local part of the address | Check the username on other platforms |
| Username | WhatsMyName, search across social networks and forums |
| Google ID | Google Maps, Calendar, archives |
| Phone (including masked) | Breaches, messengers, social networks |
| Photo | Reverse image search |
| Domain | WHOIS, DNS, subdomains, other addresses on the domain |
| Account metadata (IDs, dates) | Cross-referencing with breaches and archives |

**Confidence scale.** Label every finding:

- **Confirmed:** at least two independent sources, with matching unique attributes (photo, ID, phone, writing style).
- **Probable:** one strong source plus circumstantial indicators.
- **Hypothesis:** a single match that requires verification.

**Common mistakes:**

- **Confirmation bias:** looking for evidence for your own version and ignoring contradictions.
- **Confusing owners:** reused addresses, shared usernames, registrations with someone else's address.
- **Stale data:** the account was deleted, or the address passed to a different owner.
- **Trusting a single source**, especially breach aggregators.

**Document everything.** For each finding, record the source, date and time, the tool and its version, a screenshot and an archived copy, and separate facts from conclusions. This makes the result reproducible and protects you if your methodology is scrutinized.

---

## OPSEC: Protecting the Researcher and the Result

In email OSINT, OPSEC serves three purposes: **not revealing your interest to the target, not linking the research to your own identity, and not damaging the reputation of your IPs and accounts.**

- **Separate your environments.** Use a dedicated browser profile or virtual machine, and dedicated research accounts and email. Never sign in with personal accounts: Google-class tools use cookies, and some platforms show the owner who viewed a profile.
- **Passive before active.** Every active request leaves a trace and can lead to the target being notified.
- **Network.** Use a dedicated VPN or a separate channel for research traffic, without mixing it with personal traffic. Tor provides anonymity, but its exit nodes are often blocked or served CAPTCHAs, which causes some tools to stop working. Choose based on the task.
- **Do not scale from a single source.** Bulk checks from one IP lead to blocks and make you conspicuous. Cache results and do not repeat queries.
- **Do not contact the target.** Do not write to the address, do not open links you receive, and keep tracking pixels in emails in mind.
- **Keep an action log.** What was run, when, and with what. This is needed both for reproducibility and for legal protection.

---

## The Defensive Perspective: How to Reduce Your Own Email Footprint

The same techniques work in reverse. If you want to understand and reduce your own exposure:

- **Use different addresses and aliases for different services** (plus tags, alias services, your own domain). This severs the links between accounts and shows where a leak came from.
- **Do not make your primary address the public login** for everything, and keep the backup address separate.
- **Review your Google profiles:** hide public reviews and photos on Maps, close calendars, check privacy settings.
- **Check your addresses periodically** in Have I Been Pwned and subscribe to notifications about new breaches.
- **Enable multi-factor authentication**, preferably with an app or a hardware key rather than SMS. Use a password manager and unique passwords.
- **Limit recovery hints:** keep backup details up to date and do not attach a phone number unless it is necessary.
- **For organizations:** configure `SPF/DKIM/DMARC`, monitor breaches for the corporate domain, and train employees not to use work email for third-party services.

---

## Tool Summary

| Task | Tool | Type |
|---|---|---|
| Address validation | emailvalidator.io | Semi-active (via a third party) |
| Google account | Epieos, GHunt | Passive / active (depending on method) |
| Avatar and profile by hash | Gravatar | Passive |
| Accounts by email | Blackbird, Holehe | Active |
| Accounts by username | WhatsMyName, Blackbird | Active |
| Breaches | Have I Been Pwned, Intelligence X, Leak-Lookup | Passive |
| Public mentions | Google, Bing, Yandex, Wayback Machine | Passive |
| Domain and infrastructure | WHOIS/RDAP, DNS, crt.sh | Passive |
| Git and metadata | GitHub, `exiftool`, PGP keyservers | Passive |

> Tools and service endpoints change quickly. Before you begin, check that the tool and its documentation are current.

---

## Conclusion

An email address is not just a means of communication but one of the most stable and informative identifiers in the digital environment. Even with minimal starting data, it can lead to services, breaches, public mentions, and other traces from which a profile is gradually assembled.

But a mature approach is distinguished not by the number of tools but by discipline:

1. **Method matters more than tools.** No single technique gives an answer on its own. The answer comes from correlating independent sources.
2. **Every finding is a hypothesis** until it has been confirmed and given a confidence rating.
3. **Passive before active.** Active methods leave traces, can expose the investigator's interest, and require authorization.
4. **OPSEC and legality are part of the methodology.** Environment separation, an action log, and an understanding of the legal framework are as important as the data itself.
5. **Knowing the attack is the basis of defense.** What the researcher sees, the attacker sees too, so the same techniques are worth applying regularly to yourself and your organization.
