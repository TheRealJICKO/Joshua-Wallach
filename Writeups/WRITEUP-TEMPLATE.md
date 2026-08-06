# [Machine / Room / Topic Name]

| | |
|---|---|
| **Platform** | TryHackMe / HackTheBox (retired) / PortSwigger / local lab |
| **Difficulty** | Easy / Medium / Hard |
| **OS** | Linux / Windows |
| **Date completed** | YYYY-MM-DD |
| **Key techniques** | `SQLi` `SUID abuse` `token impersonation` |

---

## TL;DR

Two or three sentences. What was the vulnerability chain, and what did it get you?

> Example: An unauthenticated file-upload endpoint accepted `.phtml`, giving a
> web shell as `www-data`. A `sudo` misconfiguration on `tar` allowed arbitrary
> file read, exposing an SSH key for root.

---

## Recon

What did you scan, what came back, and — importantly — **what did you decide to
look at first and why?** Anyone can paste `nmap` output. The reasoning is the
part that shows judgement.

```bash
nmap -sC -sV -oN nmap/initial 10.10.x.x
```

| Port | Service | Notes |
|---|---|---|
| 22 | OpenSSH 8.2p1 | Standard, park it |
| 80 | Apache 2.4.41 | Interesting — enumerate |

## Enumeration

Directory brute-forcing, parameter discovery, version checks. Show the dead ends
too — one or two lines on what *didn't* work is honest and more useful to a
reader than a clean narrative where every guess lands.

## The vulnerability

Name the class (not just the CVE): "unauthenticated arbitrary file upload",
"second-order SQL injection", "insecure deserialisation". Explain what the
application was doing wrong in one paragraph, before you show any exploitation.

## Exploitation

Reproducible steps. Real commands, real requests. Redact target-specific
identifiers if there's any doubt.

```http
POST /upload HTTP/1.1
Host: target
Content-Type: multipart/form-data; boundary=---
...
```

Screenshots only where they add something a code block can't.

## Privilege escalation

Enumeration → the finding → the escalation. Say which enumeration step actually
surfaced it (`linpeas`, manual `find / -perm -4000`, reading a config), so the
reader learns the *method*, not just the answer.

## Why it worked

**This is the section that separates a write-up from a walkthrough.** What was
the underlying design or configuration mistake? What assumption did the
developer make that turned out to be false? Would this survive contact with a
real, patched, monitored environment — and if not, why not?

## Remediation

How would you fix it, written as if for the team that owns the system:

- **[Finding]** — [specific, actionable fix. Not "sanitise input" — say
  *validate the file extension server-side against an allowlist and store
  uploads outside the webroot*.]
- Note any compensating controls that would have caught or contained this.

## Lessons learned

What did *you* get wrong or slow on? Where did you burn time? What are you going
to look for earlier next time? Keep this honest — it's the section that shows
you're building methodology rather than collecting flags.

## References

- [Vendor advisory / CVE]
- [PortSwigger Web Security Academy — relevant topic]
- [HackTricks — relevant page]

---

<sub>Written up for my own learning. All testing performed against systems I am
explicitly authorised to test.</sub>
