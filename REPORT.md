# REPORT — what the measurement showed, who it affects, and whose responsibility it is

## 1. Measurement read-out

**What was run.** The bare agent (`Qwen/Qwen2.5-3B-Instruct`, greedy decoding, Colab T4 GPU) screened 200 matched
pairs: 40 pairs for each of the three attributes and 40 for each of the two floors. That is 400 profiles and 400
model calls (84,409 prompt tokens, 11,122 new tokens, 481.3 seconds). 0 of the 400 replies were unparseable, and
356 of the 400 profiles were shortlisted. The agent is very generous: it rejected only 44 profiles.

**Flips, read against the floor.**

| attribute | flips / pairs | flip rate [95% CI] | its floor | floor flips | intervals overlap |
|---|---|---|---|---|---|
| name_language | 2/40 | 5.0% [1.4%, 16.5%] | null_name | 0/40 (0.0% [0.0%, 8.8%]) | yes |
| gender_coded_name | 4/40 | 10.0% [4.0%, 23.1%] | null_name | 0/40 | yes |
| university | 1/40 | 2.5% [0.4%, 12.9%] | null_university | 1/40 (2.5%) | yes |

All flips were between shortlist and reject (no "other" flips). No attribute stands clear of its own floor, so this run
**does not show** that the language of the name, the gender coding of the name or the university changed the
decision. That is the main result, and I do not claim more than it.

**Direction.** Each pair was turned round so that the named group comes first:

- Gender-coded name: all 4 flips went the same way. The female-coded member was shortlisted and the male-coded
  member was rejected in 4 pairs, and the reverse happened in 0 (exact McNemar p = 0.125). Over all 80 profiles of
  these pairs, 90.0% of female-coded and 80.0% of male-coded profiles were shortlisted. The score gap is small and
  not different from zero: male minus female -0.4 points [-1.2, +0.6], sign test p = 0.219.
- Name language: the Kazakh-named member was favoured in 2 flips and disfavoured in 0 (p = 0.500). Score gaps are
  +0.7 [-1.8, +3.8] for Kazakh, +0.2 [-1.7, +2.1] for Russian and -1.0 [-4.2, +1.9] for English names, all
  around zero.
- University: only 1 flip (flagship favoured, p = 1.000), but the scores lean one way. Flagship minus regional is
  +2.5 points [+1.1, +4.1], and all 10 university pairs whose scores differ favoured the flagship (sign test
  p = 0.002). Mean scores were 63.2 (flagship) and 60.8 (regional).

**Size of the score changes (cannot cancel).**

| attribute | pairs with a score difference | mean abs. difference [95% CI] | floor | floor pairs with a difference | floor mean abs. difference |
|---|---|---|---|---|---|
| name_language | 11/40 | 3.12 [1.38, 5.25] | null_name | 7/40 | 2.00 [0.62, 3.62] |
| gender_coded_name | 6/40 | 1.12 [0.25, 2.13] | null_name | 7/40 | 2.00 [0.62, 3.62] |
| university | 10/40 | 2.50 [1.12, 4.12] | null_university | 9/40 | 2.12 [0.88, 3.62] |

On size, too, every attribute overlaps its floor. Gender is even below it. The university result is therefore of a
special kind: its **size** is like the floor, but its **direction** is not random (10 out of 10). The floor cannot
show a direction, because both of its universities have the same tier. So the honest reading is: the model gives
flagship applicants about half a score step more (it scores in steps of 5), this moved one decision in 40 pairs, and
it would matter only for applicants near the cut-off. Several tests were run (five orientations, two tests each),
so a single p-value should be read with care; p = 0.002 would still pass a Bonferroni correction for ten tests
(0.005), the gender result (p = 0.125) would not.

**What 40 pairs could detect.** An exact two-sided McNemar test at 0.05 needs at least 6 one-sided flips out of 40
(15.0%) when nothing goes the other way, 8 (20.0%) with one, and 10 (25.0%) with two. The gender result (4 to 0) is
two flips short of that. So a real gender effect of about one decision in ten could exist and this run could not
tell it from chance. Score shifts smaller than the half-width of the interval could not be told from zero either:
about ±0.9 points for gender, ±1.5 for university and ±1.9 to ±3.1 for the name-language comparisons. "No measured
effect" here means "no effect larger than this run can see", not "no effect".

**Where the decisions sit.** Shortlisted profiles scored 45-85 (median 75), rejected ones 20-45 (median 40). Score 45
is the only value that got both decisions (31 shortlisted, 3 rejected). Every one of the 8 flipped pairs (the 7 real
ones and the 1 floor flip) had at least one member scored 40, 45 or 50, and 7 had both. Any edit, relevant or not,
can only change a decision for applicants in this narrow zone.

**What the floors cover and what they do not.** `null_name` changes which name of the same language group and
gender coding is shown; `null_university` changes which university of the same tier is shown. Both keep everything
else fixed: the prompt wording, the field order, the merit values and the model. So the floors measure how much the
output moves when these particular irrelevant tokens change. They do not measure sensitivity to the wording of the
system prompt, to the layout of the profile, to another model or another GPU. I ran the optional order probe to see
how large one of these untested sources is: 40 pairs of *identical* applicants, the second shown with the same lines
in another order, gave 10 flips out of 40 = 25.0% [14.2%, 40.2%], all in one direction (shortlisted in the default
order but not in the other: 10, the reverse: 0, exact McNemar p = 0.002), and the score fell by 6.6 points
[-9.9, -3.5] in the other order. So the order of the lines moved this agent more than any protected attribute did.
This is not a floor for the attributes (their two members are laid out alike), but it shows that the agent's
decisions are fragile in ways the floors do not see. Greedy
decoding means there is no sampling noise to average; the "noise" here is the model's sensitivity to small changes in
the text. A stronger estimate would need many more pairs per attribute (to detect a 10% one-sided flip rate with
reasonable power you need well over 100 pairs), several prompt wordings, and more than one name per group in each
pair.

**What this run cannot support.** The profiles, names and universities are invented; real applicants have more
varied profiles, and many more of them would sit near the cut-off if the fund were less generous. A different prompt
or model could move any of these numbers. The result is about this agent with this prompt on these 400 profiles.

## 2. Who is affected

Everything below is tied to what the run showed. Because every flip in Part 1 happened in a narrow score zone
(40-50), the people at risk are not "some groups" in general but **applicants whose merit puts them near the cut-off**,
and inside that zone, the ones on the wrong side of a name, a university or a line order.

**Who is wrongly rejected.**

- *Applicants with a male-coded name near the cut-off.* In all 4 gender flips the male-coded twin was rejected and
  the female-coded twin was shortlisted with the same merit. Over the 80 profiles of these pairs, 80.0% of male-coded
  profiles and 90.0% of female-coded ones were shortlisted. This is not a proven effect (p = 0.125), but if it is real,
  this is the group that loses.
- *Applicants from regional universities.* They got 2.5 points less than identical flagship applicants
  (10 of 10 differing pairs), and in one pair the regional applicant was rejected while the flagship twin was
  shortlisted at the same score of 45. For most profiles 2.5 points change nothing; for someone at 45 they can.
- *Applicants with Russian- or English-language names* lost the 2 name-language flips to Kazakh-named twins. Too few to
  call a pattern.
- *Applicants rejected far below the band.* 11 profiles in the bare run and 24 in the harness run were rejected with
  a score in the 20-29 range. Nobody checks these, so if the model is wrong about them, nothing catches it.
- *Applicants whose profile is laid out differently.* The order probe flipped 10 of 40 identical applicants. If the
  real system ever renders applications in more than one layout (a new form, a different export), applicants could
  be rejected for formatting alone.

**Who is wrongly shortlisted, and who pays for it.** The mirror image of each case above: the female-coded,
flagship or Kazakh-named twin who was shortlisted while an equal twin was not. A scholarship fund has a limited
number of places, so a wrong shortlist is not free: it pushes someone else further down the next stage. The
flagship bonus matters most here. If a later stage ranks the shortlist by score, a regional applicant starts 2.5
points behind an equal flagship applicant, even where the shortlist decision itself did not change.

**The costs are not the same.** A wrongly rejected applicant can lose the scholarship and, for a low-income
applicant (income bracket is a field the fund cares about), possibly the chance to study; the bare pipeline gives no
reason and keeps no record, so the person cannot even object. A wrongly shortlisted applicant gains a chance they
should not have; the cost falls on the displaced applicant and on reviewer time. A wrong refusal (Part 4 measured 0
of 400 clean applicants refused) would cost an honest applicant the whole chance, because a refusal means no decision
at all.

**People the brief does not mention.**

- *Reviewers.* The harness sends 38 of 400 applicants (9.5%) to a person; with the band calibrated on the bare scores
  that would have been 109 of 400 (27.3%). Their work grows with the band, and a tired reviewer who sees mostly
  model scores may simply agree with them.
- *The fund.* It carries the legal risk (the Kazakh AI Law, Art. 6, forbids discrimination by sex, language or
  origin in AI systems) and the reputational risk if a pattern like the gender one turns out to be real.
- *Applicants who are refused before the model.* Minors and applicants without consent are now refused (Part 4).
  That is correct under the law, but they still need a human route; without it, a gifted 17-year-old simply drops out.
- *Applicants from institutions the model has never seen, and people who never apply.* The model already gives the
  invented "regional" names fewer points; a real, little-known college could fare the same. People who expect to be
  rejected by an algorithm may not apply at all, and no measurement of applicants can see them.

**If there is an effect this run could not see.** With 40 pairs, a one-sided effect needs at least 6 flips out of 40
(15.0%) to reach p < 0.05. A real effect smaller than that, for example one flip in ten pairs, could be there and
look like the gender result. The people who would bear it are the same ones as above: applicants near the cut-off
whose name or university is on the losing side.

**What compounds over time.** If the agent is used every year, the same small bias hits the same groups every year.
If regional-university applicants win fewer scholarships, fewer of them may apply next year, and if the fund ever
trains or tunes a model on its own past decisions, the bias is learned again. If the score is passed on to a later
stage (an interview list, another fund, a published ranking), the 2.5-point flagship bonus is carried with it and
nobody downstream knows where it came from. The decision log is what makes this visible later; without it, the bare
pipeline leaves nothing to audit.

## 3. Professional responsibility

**My responsibilities as the developer of the harness.**

1. *Measure before claiming anything, and report what the measurement cannot say.* I measured 400 profiles before
   and after, and I state clearly that no attribute cleared its floor, that 40 pairs could miss an effect below
   6 one-sided flips, and that the "after" table is zero by construction once name and university are hidden. I must
   not tell anyone that the harness "made the model fair" or "removed bias": the run does not show that.
2. *Tell the people who deploy it about the limits I found:* the score scale moved a lot when two lines were removed
   (166 of 400 scores changed, mean 14.01 points), the band calibrated on one setup sent 9.5% instead of 27.3% to a
   person in the other, the model reacts strongly to line order (10 of 40 flips), and the refusal rule was only tested
   on invented names.
3. *Build checks that run, not only promises.* Each refusal I describe was triggered in the submitted run (5 of 5,
   0 model calls, 5 refusal records in `decisions.jsonl`), and the check for wrong refusals ran on all 400 clean
   applicants (0 refused).
4. *Set stop conditions.* The harness should not go live if: there is no consent and objection procedure (Personal Data
   Law Art. 19-1); the band has not been recalibrated on real applications for the exact prompt and model in use;
   any attribute clears its floor in a larger test; the model, prompt or layout changes without re-running the
   matched-pair measurement; or nobody has the time to read the escalated cases.

**What this lab did, and what is left.** Done: the measurement with floors and intervals, the order probe, five
controls with tests, a decision log of 406 records, a control register whose format check passed, and legal
provisions read on adilet.zan.kz, EUR-Lex and the OECD database. Left: a bigger matched-pair sample (well over 100
pairs per attribute) and several prompt wordings; recalibrating the band on real data; a check of outcomes by income
bracket and age; testing the refusal rule on real names and programme titles; protecting the log against later edits;
and a human review of low-score rejections, which my harness does not send to anyone.

**What belongs to the institution, not to me.** The fund decides the purpose and must stick to it (Personal Data Law
Art. 7(8)); collects real consent and tells applicants that AI is used and how to object (Art. 19-1(2), AI Law
Art. 7(2)); answers objections within three working days (Art. 19-1(3)); chooses, trains and staffs the reviewers;
runs the appeal process; decides how long the log is kept and who may read it; and classifies the system's risk
level (AI Law Art. 17(1)). I can make these possible in the design, but I cannot do them.

**Where the two conflict.** Suppose the fund wants to switch off escalation because 38 reviews per 400 applications
cost staff time. I have measured what that control is for: every decision flip in Part 1 sat in the 40-50 zone. My
duty is then to say in writing what is lost, with these numbers, and to point out that the AI Law (Art. 8(1)) requires
the owner to keep control over the system. If the fund still insists, I can offer a narrower, recalibrated band (the
same rule on the blinded scores gives (45, 50), 8 of 400 = 2.0%) and a periodic re-measurement, but I should not
sign off a version with no human route at all, and I should not hide the decision in a configuration file. The same
applies if the fund asks to show the university to the model again "because it is useful": Part 1 shows the model
used it as a prestige bonus, not as context.
