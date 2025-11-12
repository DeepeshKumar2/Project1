no worries — let’s reset and do this like you’re brand-new to identity systems. i’ll explain every word first, then say what’s broken, and exactly what has to be done now, and finally the “later” plan (PTA plugin vs PTA handlers).

tiny glossary (plain words)

Active Directory (AD): Microsoft’s company address book. It stores user accounts and their passwords for Windows, email, etc.

PingDirectory (PD): Another address book (from Ping Identity). Some apps ask PD to log users in.

Password hash (non-reversible): AD does not keep your real password; it keeps a scrambled version that can’t be turned back into the real password.

Pass-Through Authentication (PTA): When PD doesn’t know your password, it can ask AD “is this password correct?” instead of checking locally.

PTA plugin: An add-on inside PD that 1) asks AD to verify the login and 2) if the login succeeds, saves that password into PD for next time (this is called write-back to PD).

PTA handler: A lighter add-on that only asks AD yes/no. It does not save the password into PD. No write-back.

Write-back to PD (local PD password): Storing the user’s password inside PD after a successful login.

Real-time sync PD↔AD: When a password is changed in one system, it quickly gets copied to the other.

Bind (LDAP bind): A login attempt at the directory level (like a low-level “sign in” test).

LDAP/LDAPS (389/636): The protocol/ports used to talk to directory servers; LDAPS is the secure (TLS) one.

Certificate / truststore (JKS): Files that let PD trust the secure connection to AD (proof you’re talking to the right server).

Base DN / DN map / search filter: “Where in AD should I look for users?” (like telling Google which folder to search).

Prod / Dev / Test: production (live), development, and test environments.

ESI: the team’s older/legacy identity/gateway piece that’s still in the mix today but will be removed later (retired).

Security Group (AWS): Firewall rules that allow or block network traffic between servers.



---

the problem (in 1 minute)

Some users exist only in AD (or at least their current password only “lives” in AD).

When those users try to log in through PD, PD should pass the check to AD (PTA).

If AD says “yes,” the PTA plugin should also save that password into PD so next time PD can handle it itself.

What’s going wrong: that pass-through step is failing (especially for users who have never logged into PD before).
Because AD won’t give us the real password (only a one-way hash), PD can’t fill in its copy unless that first pass-through succeeds. So login fails.


Think of it like this:

User → PD
  PD tries local password → fails (PD doesn't have it yet)
  PD asks AD (PTA) → SHOULD succeed → then PD saves it locally
  But right now: the "ask AD" step often fails → user can’t log in

Why is it failing? Most likely one (or more) of these simple, non-coding things:

Network/firewall rules not open from PD→AD (right ports/hosts).

Certificate trust mismatch (PD doesn’t fully trust the AD/ESI TLS certs).

“Where to search in AD” settings (base DN / mapping) not quite right.

Or a quirky product behavior in Dev that needs vendor guidance.



---

what they want to achieve (the goal)

Short term (now):
Make that PD→AD pass-through login always work so first-time AD users can log in via PD, and PD can then store their password locally (thanks to the plugin’s write-back).

Long term (after ESI is removed):
Change from PTA plugin → PTA handlers to simplify the setup. Handlers will keep asking AD for every login (no more saving passwords in PD), which is cleaner once the legacy piece (ESI) is gone.


---

exactly what needs to be done now (step-by-step, beginner style)

Do these in order. If a step fails, fix it before moving on.

1. Check network is open

Make sure the PD Dev/Test servers are in the correct AWS Security Group and can reach the AD/ESI servers on LDAPS (port 636 or 1636).

From a PD server: run a basic check openssl s_client -connect <AD_host>:636

If you see a TLS certificate and it connects: good.

If it hangs or refuses: firewall/SG needs fixing.




2. Check certificate trust

On the PD server, list the JKS truststore (the “who do I trust?” file). It must include:

Your internal root and intermediate certificates, and

The AD/ESI Dev server certificates.


The hostname you connect to must appear in the cert’s SAN list. If not, PD won’t trust it. Import or fix as needed.



3. Do a simple login (bind) test to AD

Use Apache Directory Studio (or Windows ldp.exe) from your laptop or a jump host.

Try to bind to each AD/ESI Dev server with the service account.

If any server fails while another works → it’s network/cert for that server.



4. Check PD’s “where to look” settings

In the PTA plugin inside PD, make sure the Base DN / DN map / search filter point to the actual OU where users live in AD.

Don’t mix conflicting patterns (either use a DN map or a straightforward filter). Keep it simple.



5. Turn on debug logs and test a real user

Pick an AD user who has never logged into PD.

Attempt a PD login and capture the exact error in the PD logs (time + message).

If it works once, try again to confirm PD now has a local password (the second login should work even if AD blips).



6. If configs got messy

Use the team’s Ansible playbook to reset PD’s plugin to the known-good baseline, then repeat steps 1–5.



7. If it still fails

Open a vendor support ticket with: the network checks, truststore details, plugin screenshots, and the debug log snippet.




> Success test: An AD-only user logs into PD, it works, and PD now has their password for next time.




---

the “later” plan you asked about (PTA plugin vs PTA handlers)

Why switch later?
Right now, we need the plugin because it saves the user’s password into PD after the first successful login (write-back). That’s how we “teach” PD the password.

After the old ESI piece is removed and things are cleaner, we can stop storing passwords in PD and always ask AD each time. That’s when handlers are better: simpler and fewer moving parts.

quick comparison

Feature	PTA plugin (now)	PTA handlers (later)

Checks password with AD	Yes	Yes
Saves password into PD	Yes (write-back)	No (never saves to PD)
Complexity	Higher (more settings, more that can break)	Lower (stateless yes/no check)
Best when…	You need PD to learn the password on first login	You’re okay always checking AD (no local copy)
Risk surface	Bigger (store passwords in 2 places)	Smaller (password only “lives” in AD)


Plain English:

Plugin = “Ask AD, and if it works, copy the password into PD for next time.”

Handler = “Always ask AD; never copy the password into PD.”


So the sentence you quoted means:

> “Later, when the old ESI part is gone, stop using the plugin and start using handlers. Handlers are simpler because they don’t try to save the password inside PingDirectory.”




---

super-short recap

What’s broken: PD’s “ask AD” step (pass-through) often fails for first-time AD users, so PD never learns their password and the login fails.

Fix now: open network ports, fix certificate trust, make sure PD searches the right place in AD, reproduce with logs, reset config if needed, and call vendor if it still fails.

Later: switch to PTA handlers (always ask AD, don’t store passwords in PD) once the legacy ESI piece is retired.
