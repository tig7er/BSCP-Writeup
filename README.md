# BSCP Exam Writeup

[![PortSwigger](https://img.shields.io/badge/PortSwigger-Web%20Security%20Academy-FF6633)](https://portswigger.net/web-security)
[![Certification](https://img.shields.io/badge/Certification-BSCP%20Passed-success)](https://portswigger.net/web-security/certification)
[![Status](https://img.shields.io/badge/Status-Passed-brightgreen)]()

## 📌 Overview

This repository contains my full writeup for the **Burp Suite Certified Practitioner (BSCP)** exam by [PortSwigger](https://portswigger.net). I passed the exam and have documented my approach, vulnerability identification process, and exact exploitation steps for every stage across both exam applications.

The exam consists of **2 web applications**, each split into **3 progressive stages**:

1. Anonymous access → Regular user session
2. Regular user session → Administrator session
3. Administrator session → Remote command execution

The vulnerability at each stage is randomized by PortSwigger, so this writeup is meant to demonstrate **methodology and triage**, not a fixed "answer key."

## 📄 Full Writeup

👉 [Read the complete writeup here](./BSCP-Exam-Writeup.md)

## 🛠️ Vulnerabilities Covered

| App | Stage 1 | Stage 2 | Stage 3 |
|-----|---------|---------|---------|
| App 1 | Username Enumeration | CSRF (Token Not Validated) | Blind OS Command Injection |
| App 2 | Host Header Poisoning | SQL Injection (Time-Based Blind) | Server-Side Template Injection (Jinja2) |

## 🧰 Tools Used

- Burp Suite Professional
- Param Miner
- Burp Collaborator
- Burp Intruder

## 🙏 Acknowledgements

- **PortSwigger** — for the Web Security Academy, the best free practical resource for learning web application security.
- **Almadad Ali** and **Avinesh Jangir** — for their guidance and support throughout this preparation.

## 📬 Connect

**Author:** Dhruv Sharma

If you found this writeup helpful, feel free to star ⭐ this repo or connect with me on LinkedIn.

---

*This writeup reflects my personal exam attempt. Vulnerabilities and exam content are randomized by PortSwigger, so your exam experience may differ.*
