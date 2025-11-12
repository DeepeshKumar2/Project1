perfect — here’s the DEV-only, UI-only, click-by-click runbook with a clear Why: before every action (just like the example you pointed out). Follow in order. Don’t touch Prod.


---

0) Backup first (DEV)

Why: if anything looks off, you can instantly roll back to today’s config.

Click path:

1. Open PingData Console (DEV) → https://<PD-DEV-HOST>:8443/console → log in.


2. Top-right ⚙ Settings → Export Configuration → Export → download the ZIP.



(Optional) Keep Ansible Tower/AWX open at Templates → locate the job that re-applies Master Plugins – DEV (easy reset if needed).


---

1) Make sure network rules allow PD→AD over LDAPS (DEV)

Why: pass-through can’t work if PD cannot reach AD/ESI on secure LDAP ports (636/1636).

Click path (AWS Console):

1. EC2 → Instances → select your PD DEV instance → Security tab.


2. Click the PD DEV Security Group (SG).


3. Outbound rules: ensure TCP 636 (and 1636 if your env uses it) to AD/ESI DEV subnets/SGs is allowed.


4. Go back to Instances, open each AD/ESI DEV instance → Security tab → its SG.


5. Inbound rules: ensure TCP 636/1636 from the PD DEV SG (or PD subnet) is allowed.


6. If you use Network ACLs: VPC → Subnets → (choose) → Network ACLs → confirm both directions allow TCP 636/1636 between PD DEV and AD/ESI DEV subnets.




---

2) Verify AD is healthy with a GUI bind (no shell)

Why: confirms AD accepts the service account and rules out AD-side issues.

Click path (Apache Directory Studio):

1. Open Apache Directory Studio → Connections → New Connection (plug icon).


2. Name: AD-DEV-1 → Hostname: <AD-DEV-HOST-1> → Port: 636 → check Use SSL (LDAPS) → Next.


3. Authentication:

Bind DN or user: svc-ping@yourdomain.com (from the spreadsheet)

Password: (from the spreadsheet)

Click Check Authentication → expect Success → Finish.



4. Repeat for <AD-DEV-HOST-2> (name it AD-DEV-2).



(If one host fails while the other succeeds, that node needs its SG/NACL/cert fixed in Step 1 & Step 3.)

(Alternative, Windows only: ldp.exe → Connection → Connect… (server=<AD-DEV-HOST>, port=636, SSL=✓) → Connection → Bind… (UPN + password) → expect “Authenticated”.)


---

3) Trust the AD/ESI DEV certificates in PD (UI only)

Why: even with network open, PD will refuse LDAPS if it doesn’t trust the AD/ESI DEV certificates. The meeting said DEV already has internal root/intermediate; we’ll verify and import ESI/AD DEV certs if missing.

Click path (PingData Console):

1. Left menu Security (or Encryption & Certificates).


2. Open Trust Manager Providers (or Certificates / Trust Stores).


3. Click the provider used for outbound SSL (often File-Based Trust Manager Provider pointing at the default truststore).


4. Click Add/Import Certificate:

If you have PEM files for AD/ESI DEV or their issuing CA, Upload/Paste → Save.

If offered, choose Import from Server → enter <AD-DEV-HOST-1>:636 → Fetch/Import → Trust → Save; repeat for <AD-DEV-HOST-2>.



5. Confirm you now see:

Internal Root & Intermediate CAs, and

AD/ESI DEV server cert(s) (or their CA).



6. If the PTA plugin lets you select a Trust Manager Provider, point it to this one (else it uses the server default you just updated).


7. Save/Apply (accept any prompt to restart affected components).



Why note: Certificate SAN must include the exact hostname you’ll configure below. If not, use the name that’s present in SAN or fix the cert.


---

4) Configure the Pass-Through Authentication (PTA) plugin (DEV)

Why: the plugin tells PD how to ask AD, and (for now) to write the password back into PD after success.

Click path:

1. Servers → choose your PD DEV server → Plugins.


2. Click Pass-Through Authentication Plugin → Edit.



A) LDAP Servers (targets)
Why: where PD will send the pass-through checks.

Add LDAP Server →

Hostname: <AD-DEV-HOST-1>

Port: 636

Connection Security: SSL/TLS

Search/Bind account (if shown):

User: svc-ping@yourdomain.com

Password: (service account password)



Add <AD-DEV-HOST-2> the same way → Save.


B) User Lookup (Search Base + Filter)
Why: tells PD how to find the user in AD. Keep it simple (don’t mix DN maps with search filters).

Search Base DN: dc=yourdomain,dc=com (your AD base)

Search Filter: choose one based on what users type when logging in:

Short username → (sAMAccountName={user})

Email/UPN → (userPrincipalName={user})



C) Behavior toggles
Why: matches the meeting’s intent—PD tries local first, then AD, and saves the password back to PD on success while ignoring PD’s complexity if AD already approved it.

Try local password first → Enabled

Update/override PD password on successful pass-through → Enabled

Relax PD password policy for pass-through → Enabled


Click Save.
(If you see “save timed out,” close and reopen—if values are there, it saved, as noted in the meeting.)

D) Apply
Why: cleanly re-initializes the plugin with your changes.

Back on Plugins → toggle Pass-Through Authentication Plugin: Disable → Save → Enable → Save.



---

5) Test the exact scenario (first-time AD user) with GUI only

Why: this is the case they were stuck on—an AD user who never logged into PD should work now; a second login proves PD stored the password locally.

Click path (Apache Directory Studio):

A) Find the user’s PD DN (to bind as that user)

1. Create a PD DEV browsing connection:

Name: PD-DEV-BROWSE

Hostname: <PD-DEV-HOST> | Port: 636 | Use SSL ✓ → Next

Authentication: Bind DN = your PD admin (e.g., cn=Directory Manager) + password → Check Authentication → Finish.



2. Expand the tree → browse to your people/users OU (e.g., ou=people,dc=yourorg,dc=local).


3. Use the search field to find your test user (one who has never logged into PD).


4. Right-click the entry → Copy DN (this is the user’s Bind DN in PD).



B) Log into PD as that user — twice

1. New Connection:

Name: PD-DEV-USER-TEST

Hostname: <PD-DEV-HOST> | Port: 636 | Use SSL ✓ → Next

Authentication:

Bind DN or user: paste the user DN from step A

Bind password: the user’s current AD password

Click Check Authentication → expect Success → Finish.




2. Disconnect that connection, then Connect again (second login).

Expect Success again (this proves the write-back to PD happened on the first login).





---

6) If it fails in DEV, collect proof via UI

Why: the team’s next step was to enable debug briefly and (if needed) open a vendor ticket with clear artifacts.

Click path (PingData Console):

1. Logs → Error Log / Server Error Logger → Edit.


2. Turn on Include Debug Messages (or set level to include Debug) → Save.


3. Repeat the first-time login test (Step 5B). Note the timestamp.


4. Return to Logs → copy the error lines around that time (look for “pass-through auth unsuccessful”).


5. Set log level back to Normal (turn Debug off).



(If DEV configs feel messy, run the Ansible “Master Plugins – DEV” job to reset, then redo Steps 3–5.)


---

7) Institutional/school users (DEV validation only)

Why: per the meeting, don’t spend time on edge cases here; plan to force a password reset for this population instead of relying on PTA.

Click path (DEV only):

1. Plugins → Pass-Through Authentication Plugin.


2. Find any mapping/policy specifically for institutional/school users.


3. Disable that mapping/rule in DEV → Save (this is just to validate approach; don’t touch Prod yet).




---

8) “Done” checklist (DEV)

Why: confirms you’ve met the meeting’s goal without touching Prod.

✅ In Directory Studio, AD bind succeeds for AD-DEV-1 (and AD-DEV-2, or you fixed its SG/cert).

✅ In PingData Console, AD/ESI DEV certs (or CA) are trusted.

✅ PTA plugin is set with (a) both AD servers, (b) Base DN + one simple Filter, and (c) the three Behavior toggles Enabled.

✅ A never-logged AD user can log in to PD (DEV): first login succeeds, second login also succeeds (write-back confirmed).

✅ If anything still blocks you, you’ve got a debug log snippet, screenshots, and GUI bind screenshots ready to send to vendor support.



---
