# JUSTIFICATION — the design of the harness, and why

The chain this document follows for each control is: **measurement (Part 1) → risk → ethical principle → legal
constraint → control in code → check in the submitted run.** All numbers come from the executed notebook
`lab06_harness_says_no.ipynb`; the code of the controls is in `lab06_core.py` (`validate_profile`,
`refusal_reason`, `screen`, unchanged) and in the notebook cells of Part 2 (`BLIND`, `my_extra_refusal`,
`my_extra_escalation`, `BORDERLINE`, `HARNESS`).

| control | measurement → risk | principle | legal constraint (read on adilet.zan.kz) | where in code | checked in the run |
|---|---|---|---|---|---|
| `validate_input` | Part 1 used only clean profiles; the bare pipeline rejected GPA 9.9 / exam -5 as if on merit (Part 4) | Robustness, security and safety (OECD 1.4) | AI Law Art. 15(2)(1)-(2) | `validate_profile`, step 1 of `screen` | Part 4: `invalid_input`, 0 model calls |
| `blind` (name, university) | 4/0 gender flips (p = 0.125); flagship +2.5 points, 10/10 (p = 0.002); none above its floor | Fairness and non-discrimination (OECD 1.2(a), UNESCO 28-30) | AI Law Art. 6; Constitution 2026 Art. 16(2); PD Law Art. 7(9) | `BLIND`, step 3 of `screen` | Part 3: 401 records with name and university withheld |
| `refuse` (+ `my_extra_refusal`) | bare pipeline shortlisted "rank by ethnicity" and an injected name (Part 4) | Safeguards against misuse (OECD 1.2(b)); do no harm (UNESCO 25-26) | PD Law Art. 7(1), 7(8), 19-1(1); Civil Code Art. 17(1), 22(1); AI Law Art. 15(2)(2) | `refusal_reason`, `my_extra_refusal` | Part 4: 4 refusals by this control, 0 model calls; 0/400 wrong refusals |
| `log` | bare pipeline kept no record of its 400 decisions | Accountability and traceability (OECD 1.5, UNESCO 42-43) | PD Law Art. 19-1(2)-(3), 24(1)(1); AI Law Art. 8, 15(2)(3), 16(1)(4)-(5) | `screen` → `finish`, `decisions.jsonl` | 406 records, one per call |
| `escalate` (+ `my_extra_escalation`) | all 8 flips of Part 1 had a member scored 40-50 | Human oversight and determination (UNESCO 35-36, OECD 1.2(b)) | AI Law Art. 8(1), 9; PD Law Art. 19-1(2)-(3) | `BORDERLINE = (40, 50)`, step 6 of `screen` | Part 3: 38 of 400 escalated (9.5%) |

## 1. What the system may do

**Purpose, stated narrowly.** The system may read one scholarship application of an adult who consented to automated
processing and give the Tulpar fund a *recommendation*: shortlist or reject, with a score and a short reason. It may
do this only for the purpose `scholarship_screening`. It may not rank applicants by any personal characteristic,
build profiles over time, or be used for any other purpose (Personal Data Law Art. 7(8); AI Law Art. 17(3)(3)).

**What it decides alone and what it only informs.** In the submitted harness the model's decision stands on its own
only outside the band 40-50 and only when the reply parsed and did not trip my escalation rule: in Part 3 that was 338
shortlists and 24 rejections out of 400. Everything in the band (38 of 400), every unparseable reply and every case my
rule flags goes to a person, and every refusal is made by fixed code, never by the model. Ethically (UNESCO para 26)
and under GDPR Art. 22 as a benchmark, a rejection should not be fully automatic; my design still lets the 24
low-score rejections stand without review, and I list this as an open gap in section 6.

**Which constraint made me build each control.** See the table above. Two of them changed the architecture itself:

1. *Refusals happen before the model, in plain code.* Personal Data Law Art. 19-1(1) forbids automated processing that
   affects someone's rights without consent, and Art. 7(1) needs consent of the subject or the legal representative.
   If the law only asked me to "be careful", I would have let the model see the profile and flag doubtful cases
   afterwards. Because of these articles, a profile without consent or of a minor must not reach the model at all,
   so the refusal check runs first and makes 0 model calls (Part 4: 5 refusals, 0 calls).
2. *Blinding instead of a counterfactual check.* AI Law Art. 6 excludes discrimination by sex, language and origin.
   I could have kept the name and re-asked the model with a swapped name (the optional `counterfactual_check`), but
   that doubles the model calls and still shows the model the name. Blinding meets the same constraint at zero extra
   calls (400 calls before and after), so I chose it and left `counterfactual_check` off.

A third, smaller change: the program field is still shown to the model even with blinding, so the refusal rule
checks free-text fields (name, university, program) rather than only the fields the model sees. A name that is an
instruction does not reach this model, but it would reach a human reviewer's screen and any later system.

## 2. What it must refuse

| refusal (reason code) | enforced in | fired in the submitted run | legitimate case it also blocks | who bears the cost |
|---|---|---|---|---|
| `purpose_not_allowed` | `refusal_reason` (purpose not in `ALLOWED_PURPOSES`) | Part 4, record `R73e4e90179d3` | an approved fairness audit by the fund that needs ethnicity data | the fund's auditors, who must use a separate, approved process |
| `minor` (age < 18) | `refusal_reason` | Part 4, record `Rf014420fa054` | a gifted 17-year-old who enters university early | that applicant, who needs a guardian's consent and a manual route |
| `no_consent` | `refusal_reason` | Part 4, record `Rce4af60afee3` | an adult who wants to apply but not to be judged by AI | that applicant, unless the fund offers manual screening |
| `invalid_input` | `validate_profile` (step 1 of `screen`) | Part 4, record `R7b76ad0d571c` | an applicant whose record has a typing error (GPA 3.9 typed as 39) | that applicant, who must be told and allowed to fix it |
| `prompt_injection_suspected` | `my_extra_refusal` (instruction words in name, university or program) | Part 4, record `R2fcf2c98a088` | a real programme or name containing "Score", "Output" or "Prompt" | that applicant; I kept the word list short to limit this |
| `malformed_text_field` | `my_extra_refusal` (digits, colons, newlines, braces, empty or >80 characters) | not triggered in the run | a name with a digit or an unusual sign, a programme like "Law: International" | that applicant |

The `malformed_text_field` code was tested offline while I wrote the rule (for example a name with a newline and a
fake "GPA" line is refused) but **it did not fire in the submitted run**, so for this run it is a tested rule, not an observed one. In
the 400 clean measured applicants the refusal rules (built-in and mine together) refused 0.

**Is the cost acceptable?** For purpose, minor and no consent, yes: the law (PD Law Art. 7(1), 7(8), 19-1(1); Civil
Code Art. 17(1), 22(1)) does not let the fund process these automatically, so the cost exists anyway; the fair answer
is a manual route, not an automatic decision. For the two text rules, the cost is a real risk to honest applicants
with unusual names, and 0 of 400 on invented names is weak evidence. I accepted it because a refused applicant can be
handled by a person, while a successful injection would change a decision silently.

**What happens to a refused request.** It is recorded: one log line with outcome `refuse`, the reason code, the
control that fired, and `raw_output` starting with `[no model call]` (5 such records in `decisions.jsonl`). Nobody is
told automatically and there is no built-in "ask for a human decision" button: that is the fund's part (notice,
objection within three working days under PD Law Art. 19-1(2)-(3), and a manual route). The record makes it possible.

## 3. What record it keeps

**The audit question.** *Why did this applicant get this outcome, under which settings, and can someone check it
later without re-running the model?*

A record in `decisions.jsonl` answers it. The Part 3 record of application `P0124a` (record `Rac68ae9678c8`) says:
the model saw `program, age, gpa, exam_score, household_income_bracket, achievements, consent_to_automated_processing`
and not `name, university`; model `Qwen/Qwen2.5-3B-Instruct`, prompt `v1`, the full harness settings (band 40-50);
the raw reply `{"decision": "shortlist", "score": 60, "reason": "Average performance but consent to processing"}`;
controls `blind, log`; outcome `shortlist`, reason code `model_decision`, not escalated; timestamp
`2026-11-21T11:40:27Z`; and the SHA-256 of the exact text the model saw. A refusal record says which rule stopped
the request and that the model was never called.

**What is in it and what is left out on purpose.** Each record has 16 keys (`RECORD_KEYS`): record id, application id
(`pid`), input hash, fields shown and withheld, model id, prompt version, harness settings, raw reply, decision, score,
controls fired, outcome, reason code, escalated flag and timestamp. It holds **field names, not field values**: no
name, no university, no grades. I chose this line because the log is copied and kept longer than the application
itself, and the Personal Data Law asks that data not be excessive for the purpose (Art. 7(9)). The `pid` links the
record to the application database, which already has the values. The raw reply can quote values the model was shown
(here: none of the blinded fields).

**Keeping, reading, changing.** Kazakh law did not give me a fixed retention period for this kind of record in the
articles I read, so I propose: keep each record for the scholarship cycle plus the appeal period and at least six
months (the EU AI Act, Art. 26(6), used only as a benchmark, asks deployers for at least six months). Reading should
be limited to the review officer, the data-protection officer and auditors (PD Law Art. 22(1): prevent unauthorised
access). Nobody should change it. Today nothing technical stops an edit: the file is plain JSONL. The record id is a
hash of the record and its line number, so an edited line would no longer match its id, but nothing checks this, and
whoever edits the line can compute a new id too. Real protection would need append-only storage or a hash chain
(each record holding the hash of the one before), which I did not build.

**What the record does not let you reconstruct.** The applicant's actual data (by design; you need the database and
the `pid`); the exact prompt text (only its hash, which you can check if you re-render the profile); why the model
chose its score (the `reason` is one sentence written by the model and is not a faithful explanation: "Average
performance but consent to processing" explains little); what a human reviewer later decided on an escalated case
(the reviewer's decision is not logged); and the bare runs of Part 1 and Part 4, which were not logged at all.

## 4. The legal constraints

**Which law applies, and why.** The fund is described as operating in Kazakhstan, and the system makes automated
decisions about natural persons using their personal data. So the instruments that would bind a real deployment are
Kazakh. I read each provision below on the official database adilet.zan.kz (Russian text) on 21 November 2026.
Both laws bind any "owner" or "operator", private or public, so they would bind the fund even if it is private. The
prototype itself only processes synthetic data, so here these laws are applied **voluntarily**, as the rules the
system would have to meet on launch.

1. **Law of the RK No. 94-V of 21 May 2013 "On Personal Data and Their Protection"** (in force; last amended
   14.09.2026; https://adilet.zan.kz/rus/docs/Z1300000094).
   - **Art. 19-1(1)** (added by Law No. 231-VIII of 17.11.2025): automated processing of personal data that creates,
     changes or ends a person's rights or legitimate interests is prohibited, except with the subject's consent or
     where a law provides for it. *Requires:* without consent, no automated shortlist/reject at all. This is the
     provision that is written specifically for automated decisions. → the `no_consent` refusal.
   - **Art. 19-1(2)-(3)**: the operator must explain the procedure of automated processing and its possible
     consequences, let the person object, and consider the objection within three working days and tell the person
     the result. *Requires:* a notice, an objection channel and a record good enough to answer the objection in
     three days. → the decision log and the escalation route; the notice and the objection channel are the fund's job
     and are not built here.
   - **Art. 7(1)**: data are collected and processed with the consent of the subject or of the subject's legal
     representative. **Art. 7(8)**: processing must be limited to specific, predetermined and lawful purposes.
     **Art. 7(9)**: data that are excessive for the purpose must not be processed. → the `purpose_not_allowed` refusal
     (7(8)), blinding and a log without field values (7(9)).
   - **Art. 22(1)**: the owner/operator must take measures that prevent unauthorised access and detect it in time.
     → input validation and the injection refusal are part of this; the log itself must also be protected.
   - **Art. 24(1)(1)**: the subject has the right to know the purpose, sources and methods of processing. → the log
     records which fields were shown and why each outcome happened.
2. **Law of the RK No. 230-VIII of 17 November 2025 "On Artificial Intelligence"** (in force 60 days after publication
   on 18.11.2025; last amended 25.08.2026; https://adilet.zan.kz/rus/docs/Z2500000230).
   - **Art. 6**: AI systems must be built and run so that they ensure fairness and equality and exclude any
     discrimination by origin, social and property status, sex, race, nationality, language, place of residence
     "or any other circumstances". → blinding of name (language, gender coding) and university.
   - **Art. 7(2)**: a person about whom decisions are taken with AI has the right to be informed about the automated
     processing, its consequences and the possibility to object.
   - **Art. 8(1)**: the owner, possessor and/or user must keep constant control over the AI system at all stages of its
     life cycle; **Art. 8(2)**: they bear responsibility for its results. → escalation and the log.
   - **Art. 15(2)(1)-(3)**: owners and possessors must manage risks, ensure safety and reliability (including
     protection against unauthorised access and failures) and keep documentation scaled to the system's impact. →
     input validation, injection refusal, and the log as part of the documentation. The law has no explicit "logging"
     article; I looked and did not find one, so the log rests on Art. 8 and Art. 15(2)(3), not on a logging rule. Of all five controls, the log was the hardest to tie to a specific provision.
   - **Art. 16(1)(4)-(5)**: a user may ask for an explanation of results that affect his or her rights and for the data
     on which the decision was based. → reason codes and `fields_shown` in each record.
   - **Art. 17(3)(3)**: AI systems that evaluate and classify people over time on the basis of their social behaviour
     or personal characteristics are prohibited (unless a law provides for it). A one-off scholarship screen on merit
     fields is not that, but a purpose such as "rank by ethnicity" moves towards it. → the purpose refusal.
3. **Constitution of the RK of 15 March 2026** (adopted by referendum, in force from 1 July 2026; the 1995 Constitution
   lapsed on that date; https://adilet.zan.kz/rus/docs/K2600000000), **Art. 16(1)-(2)**: everyone is equal before the
   law; no discrimination by origin, social and property status, sex, race, nationality, language or any other
   circumstance. The Constitution binds the state directly; for a private fund it works through statutes such as AI
   Law Art. 6 (my reading).
4. **Civil Code of the RK (General Part) No. 268-XIII of 27 December 1994**, **Art. 17(1)**: full legal capacity starts
   at 18; **Art. 22(1)**: minors aged 14-18 make transactions with the consent of their legal representatives. The
   Personal Data Law has no special article on minors; together with its Art. 7(1) ("or his legal representative")
   this means a 17-year-old's own consent flag is not enough. → the `minor` refusal.

**Required vs. left to us.** The law *requires* consent (or a legal basis) for automated decisions, purpose
limitation, minimisation, a way to object, an answer within three working days, human control and documentation.
It does *not* say how wide the human-review band must be, it does not require blinding as such, and it does not
require a per-decision log in JSONL form. Those are my design choices to meet the requirements. I did **more** than
the law requires in three places: the injection refusal (the law only asks for "reliability"), the
reason-text escalation rule, and the size of the review band.

**Foreign and international texts, used as benchmarks only.** The EU GDPR (Regulation (EU) 2016/679) applies to a
controller outside the EU only when it offers services to, or monitors, people who are in the EU (Art. 3(2)); the EU
AI Act (Regulation (EU) 2024/1689) applies to deployers outside the EU only when the output is used in the EU
(Art. 2(1)(c)). A Kazakh fund screening Kazakh applicants is outside both. I still used them as a benchmark because
they are more detailed: GDPR Art. 22(1) and 22(3) (right not to be subject to a solely automated decision; at least
human intervention, the right to express a view and to contest), AI Act Annex III point 3(a) (systems that decide
access or admission to education are high-risk; a scholarship is not named, so this is an analogy), Art. 12(1)
(automatic logging), Art. 14(4)(d) (the overseer must be able to disregard or reverse the output), Art. 15(5)
(resilience to inputs designed to cause mistakes) and Art. 26(6) (deployers keep logs for at least six months). The
amended AI Act (Regulation (EU) 2026/1744) applies the Annex III obligations only from 2 December 2027.

## 5. The ethical principles

I took the principles from two texts that are **not law**: the OECD Recommendation of the Council on Artificial
Intelligence (OECD/LEGAL/0449, adopted 2019, revised 3 May 2024; a non-binding Recommendation, and Kazakhstan is not
listed as an adherent) and the UNESCO Recommendation on the Ethics of Artificial Intelligence (adopted 23 November
2021, section III.2). Each one bears on a concrete decision in this system:

- **Fairness and non-discrimination** (OECD Principle 1.2(a); UNESCO paras 28-30) → what the model sees. I hid the name
  and the university because they are not merit and Part 1 showed one-sided signals on both.
- **Human agency and oversight / human oversight and determination** (OECD 1.2(b); UNESCO paras 35-36) → which
  cases a person decides: the 40-50 band, unparseable replies, and my two escalation rules.
- **Robustness, security and safety** (OECD 1.4(a)-(b); UNESCO para 27) → refusing impossible values and text that
  tries to instruct the model.
- **Transparency and explainability** (OECD 1.3; UNESCO paras 37-41) → reason codes and `fields_shown` /
  `fields_withheld` in every record.
- **Accountability and traceability** (OECD 1.5(a)-(b); UNESCO paras 42-43) → the log, with the model id, prompt
  version, harness settings and input hash.
- **Proportionality and do no harm** (UNESCO paras 25-26) → hide only what is not needed; keep all merit fields.

**Where the principles conflict.** (1) *Treating everyone identically vs. treating them fairly.* Blinding treats
every applicant the same, but it also removes the university, which a fair reader might use to put a GPA in context
(grading differs between schools). I put equal treatment first, because Part 1 showed the model using the university
only as a prestige signal (flagship +2.5), not as context. (2) *Safety vs. fairness to honest applicants.* A strict
injection rule protects the model but can refuse a real person with an unusual name; I kept the rule short and
checked it on all 400 clean profiles. (3) *Human oversight vs. privacy and cost.* More escalation means more people
read personal data and more work; I accepted it for the zone where decisions actually flipped. (4) The income bracket
is a "property status" in AI Law Art. 6, but here it is a legitimate need criterion of a scholarship, so it stays
visible; ethics (helping those in need) and a literal reading of non-discrimination pull apart, and I chose need.

**Where ethics asks more than the law.** The law accepts consent as enough for an automated rejection
(Art. 19-1(1)). Ethically, a refusal of a scholarship can change someone's education, so UNESCO para 26 (final human
determination where impact is hard to reverse) suggests a person should look at every rejection, not just the
borderline ones. My harness does not do this; it is listed as an open gap in section 6.

## 6. The trade-offs

**What each control cost, and what it bought (before = bare run, after = harness run, same 400 profiles).**

| control | cost measured | harm it prevents |
|---|---|---|
| `validate_input` | no model calls; it refused 1 request in Part 4 and 0 clean applicants | a broken record (GPA 9.9, exam -5) becoming an ordinary-looking rejection |
| `blind` | prompt tokens fell from 84,409 to 77,922; but 166 of 400 scores changed (mean 14.01 points) and 50 of 400 outcomes changed, including 7 shortlist → reject and 5 reject → shortlist; the university is no longer available as context for the GPA | the model using name (gender/language) or university prestige: the signals of Part 1 (4/0 gender flips, flagship +2.5) can no longer act |
| `refuse` | no model calls; 4 refusals in Part 4; 0 of 400 wrong refusals, but unusual real names could be refused | processing without consent, of a minor, for a forbidden purpose, or with an injected instruction |
| `log` | 406 records stored; a new personal-data store that must be protected; no model calls | decisions that nobody can explain or contest later |
| `escalate` | 38 of 400 applicants (9.5%) read by a person; 0 extra model calls | automatic decisions in the zone where Part 1 showed decisions are unstable |

Overall compute did not change: 400 model calls in both runs (1.00 per profile), 11,122 vs 11,088 new tokens, 481.3 s
vs 490.1 s of model time. The price of the harness is **people's time** (38 reviews) and **lost information** (the
university), not GPU time.

**The width of the band.** I chose (40, 50) from the bare scores: score 45 was the only value that got both decisions,
the model scores in steps of 5, and every flipped pair of Part 1 had a member at 40-50. On the bare scores it would
have sent 109 of 400 (27.3%) to a person. After blinding, the model's score scale moved: its rejections now score
20-45 and its shortlists 50-85, so the same band sent only 38 of 400 (9.5%), all as `borderline` (my own escalation
rule fired 0 times, and there were 0 unparseable replies). I did **not** retune the band on the same data I report on;
I measured the shift instead. Applying the same rule to the blinded scores gives (45, 50), which would have held
8 of 400 (2.0%). Who reads the 38: a trained reviewer of the fund, with the full application, including name and
university. What happens outside the band: 338 shortlists and 24 rejections stand without anyone re-checking them,
including the profiles rejected at 20-29. A band calibrated on one configuration does not transfer to another; before
launch it has to be set again on real applications for the exact model, prompt and layout in use.

**Why this balance.** I put fairness and the law first where they were clear (blinding, refusals) and accepted
reviewer time where Part 1 showed instability (the 40-50 zone). I would choose differently if a larger test showed an
attribute clearly above its floor (then I would also add the counterfactual check and review all rejections of the
affected group), if reviewers had no capacity (then a narrower recalibrated band, with a regular re-measurement), or
if real data showed the injection rule refusing honest applicants (then I would route those cases to a person
instead of refusing them).

**What the design still fails to protect against.**

- *Low-score rejections are fully automatic.* 24 applicants were rejected with no human look. Closing this means
  sending every rejection to a person, which the fund would have to staff.
- *Fragility to layout and wording.* The order probe flipped 10 of 40 identical applicants. My harness always renders
  the same layout, but any change in the form or the prompt can move decisions more than any protected attribute did.
  Closing this needs a fixed, tested template and a re-measurement on every change.
- *Proxies that stay visible.* The program, age and income bracket are still shown. I did not test whether they act as
  proxies for gender or origin, and the income bracket is itself a "property status" under AI Law Art. 6, kept on
  purpose as a need criterion. Closing this needs matched-pair tests on those fields.
- *Effects smaller than 40 pairs can see* (below 6 one-sided flips). Closing this needs well over 100 pairs per
  attribute.
- *The fund's side of the law is not built:* no notice to applicants, no objection channel with a three-day answer
  (PD Law Art. 19-1(2)-(3)), no manual route for refused applicants, no reviewer decision in the log, and no
  protection of the log against edits. These are outside the notebook; I describe them, I do not claim them.

## AI-use declaration

I used two AI tools in this assignment.

**Claude Code (Anthropic, model Claude Opus 5.5, in the Claude desktop app).** I used it as an assistant for most of
the practical work: to read the repository and the supplied modules, to plan the order of the work, to write the code
in the Part 2 cells (`BLIND`, `my_extra_refusal`, `my_extra_escalation`, the band, `HARNESS`, the control register and
the small evidence prints in the observation cells), and to drive Google Colab (T4 GPU) through the Claude in Chrome
browser extension, so that the notebook ran on the real model. The predictions were written into the notebook before
each run that reveals the answer, and every number in `REPORT.md` and `JUSTIFICATION.md` was copied from the
notebook's printed output. Claude Code also drafted the text of the observations, `REPORT.md` and this document, which
I then reviewed. `decisions.jsonl` and `control_register.json` were written by the notebook itself, not by hand.

**ChatGPT (OpenAI).** I used it to plan the assignment and to prepare the instructions I gave to Claude Code.

**Legal sources.** Claude Code was used to find the legal texts, but the citations do not come from a chat answer.
The Kazakh provisions were read on the official database adilet.zan.kz (the Personal Data Law No. 94-V, the AI Law
No. 230-VIII, the Constitution of 15 March 2026, the Civil Code); the key articles (PD Law Art. 7(8), 7(9), 19-1,
22(1), 24(1); AI Law Art. 6, 7(2), 8, 15(2), 16(1), 17(3), 21; Constitution Art. 16 and 94) were opened and read again
directly on adilet before they were cited. The GDPR and the AI Act were read on EUR-Lex, and the OECD Recommendation on
the OECD legal instruments database. The UNESCO Recommendation was read on unesco.org through a web-fetch tool; the
official PDF could not be opened from this machine, so the UNESCO paragraph numbers rest on that web page only.
Responsibility for the content, including any error in it, stays with me.
