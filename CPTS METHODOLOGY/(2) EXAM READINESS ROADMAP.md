

Built from the "CPTS EXAM Before do" doc + the gzzcoo 2026 review. This is the
plan that gets you from "finished the modules" to "ready to sit the exam."

---

## What the exam actually is (so you train for the real thing)

- A realistic company network, mostly Active Directory, that you attack over about
  10 days from home.
- You get a foothold on one machine (almost always through a web app), then work
  inward machine by machine until you own the domain, collecting flags along the way.
- You pass by collecting enough flags AND handing in a professional report. BOTH are
  required. Most people get the flags they need in the first 3 to 4 days and spend
  the remaining days writing the report.

## The two rules every passer repeats

1. **Enumeration, then more enumeration.** People fail from missing a port, a share,
   a user, a credential, not from a weak exploit. When stuck, enumerate more.
2. **Think dumber first.** Before complex exploit chains, try the obvious:
   admin:admin, default/weak passwords, plaintext creds in files, and password reuse
   across machines. The reviewer wasted 1.5 days by skipping the simple stuff.

---

## YOUR PLAN, IN ORDER

### Phase 1 - Finish the foundation (you are here)

- Finish all modules.
- As you finish each, add its key technique as one line into your Methodology Map
  (file 0) and one-liner into the Cheatsheet (file 0b). Grow the maps, do not reread.

### Phase 2 - Modern tool fluency (do this before boxes, it saves days)

The review is clear that specific modern tools matter. Get comfortable with these,
all driven from Linux (Windows attack box is not needed):
- **BloodHound** (map AD attack paths) + **bloodyAD** (manual AD enumeration/abuse).
- **NetExec (nxc)** (spray creds, run commands, enumerate SMB/WinRM across a subnet).
- **Impacket suite** (psexec, wmiexec, secretsdump, GetUserSPNs, GetNPUsers, etc.).
- **Ligolo-ng** for pivoting/tunneling. Learn this instead of only the older tools.
- **Sysreptor** for the report (you can start free). Learn it now, not on exam day.
(These live in your notes 12 and 13 plus this tooling list. Keep them one click away.)

### Phase 3 - The priority boxes (practice the methodology on these)

From the doc, focus your box practice here first. Do them with your Methodology Map
open, naming which stage you are in at each step:

   **TombWatcher, Administrator, Media, Trick, Heist, Snoopy, Pov, Postman**
   
Then do the rest of the HTB **CPTS Track** (https://app.hackthebox.com/tracks/76).
Some track boxes feel harder than the exam on purpose. That is the point.
Use **IppSec's CPTS playlist** when you get stuck on one: watch how he structures his
thinking, not just the answer.
(Playlist: https://www.youtube.com/playlist?list=PLidcsTyj9JXItWpbRtTg6aDEj10_F17x5)

### Phase 4 - Pro Labs (if you have time, best exam simulation)

- **Zephyr** - the top pick for mastering Active Directory.
- **Dante** - best for practicing tunneling/pivoting across several machines.
- **Offshore** - hardest, closest to the exam's complexity. A stretch goal.
If time is short, prioritise Zephyr (AD is where the exam's points are).

### Phase 5 - The dress rehearsal (do not skip this)

Do the **Attacking Enterprise Networks (AEN)** module as a MOCK EXAM:
- No walkthroughs. Just the engagement letter and your own notes.
- Take screenshots and notes exactly as you would in the real exam.
- Write a full practice report from it in Sysreptor.
This is the single best test of whether your methodology and your reporting hold up
under real conditions. Treat your performance here as your readiness signal.

---

## Exam-day strategy (keep this visible during the exam)

- Do NOT rush. Slow, complete enumeration beats fast guessing.
- Think dumber first (weak creds, reuse) on every machine.
- Once you get max privileges on any box: dump EVERYTHING (hashes, creds, configs,
  keys). Those are the keys to the next machine.
- Reuse every credential you find, everywhere, with NetExec across the subnet.
- Screenshot every step and every flag AS you go. If it is not screenshotted and
  noted, it did not happen for the report.
- Expect a slow Day 1 (enumeration), flags building Days 2 to 4, then report writing.
  A quiet day is normal, do not panic.

## The report (half the battle, literally)

- Reports run long (the reviewer's was ~170 to 200 pages, mostly screenshots + steps).
- Practice formatting during AEN, not on exam day.
- Use Sysreptor; you can highlight critical findings cleanly for a professional look.
- Your Methodology Map doubles as your report structure: per host, how you enumerated,
  got in, escalated, what you looted, each with screenshots.

---

## "Am I ready?" checklist

- [ ] Remaining modules finished, key points added to my maps.
- [ ] I can run the full AD flow (BloodHound -> roast/spray -> abuse path -> lateral
      move -> DA) from memory on an easy box.
- [ ] I can pivot with Ligolo-ng without looking it up.
- [ ] I can stabilise a shell and transfer files both ways on reflex (file 0b top box).
- [ ] I did several priority boxes + a chunk of the CPTS Track unaided.
- [ ] I completed AEN blind AND wrote a full report from it in Sysreptor.
- [ ] My note-taking produces a clean report without extra effort.
When most of these are ticked, book the exam. You do not need to feel fearless, you
need your methodology to be automatic.

---

## Resources (what each is for)
- CPTS Track (do it): https://app.hackthebox.com/tracks/76
- IppSec CPTS playlist (methodology): https://www.youtube.com/playlist?list=PLidcsTyj9JXItWpbRtTg6aDEj10_F17x5
- gzzcoo 2026 review (tips): https://blog.gzzcoo.com/posts/Certified-Penetration-Testings-Specialist-(CPTS)-2026-Review/
- CPTS 2026 tips PDF: https://www.scribd.com/document/975803984/CPTS-Review-Tips-2026
- "100/100 first attempt" write-up: https://medium.com/@hydra007/how-i-passed-the-htb-cpts-with-a-perfect-100-100-on-my-first-attempt-736226738cde
