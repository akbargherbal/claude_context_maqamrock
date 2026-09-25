# Step 2 — Sampling-Knob Probe: Listening Evaluation Checklist

**Reference config:** `c3050_a0.5` (anchor — same as Step 1/3 baseline).
**Protocol:** 1-5 holistic score + explicit bleed flag (yes/mild/no) —
score and bleed are separate axes, don't infer one from the other.
**n note:** 2 tracks per knob (Hijaz + Kurd). This is enough to ask "does
this knob move the needle at all," not enough to crown a best value off a
single delta — don't over-read one strong or weak track per knob.

Once the tracks + `KEY.json`/`KEYS.txt` land, fill in the blind filename
column from the key, then listen blind and fill in the rest.

---

## guidance_scale = 1.0 (g1.0)

- [ ] **Hijaz** — file: `Hijaz_20260924.wav`
  - Score: `3.5/5`
  - Bleed flag: ☐ no 
  - Notes: الموسيقى بسيطة ، لكن ليس هناك أخطاء تذكر
- [ ] **Kurd** — file: `Kurd_20260924.wav`
  - Score: `3/5`
  - Bleed flag: ☐ yes
  - Notes: الموسيقى جيدة لكن هناك أخطاء ترد في الموديل الأساسي.

## guidance_scale = 1.5 (g1.5)

- [ ] **Hijaz** — file: `Hijaz_20260924.wav`
  - Score: `3.5/5`
  - Bleed flag: ☐ no 
  - Notes: هناك بعض الأخطاء الموجودة في الموديل الأساسي، مثلًا يقول (سلسادها بدل من سلسالها)
- [ ] **Kurd** — file: `Kurd_20260924.wav`
  - Score: `3.5/5`
  - Bleed flag: ☐ mild 
  - Notes: الموسيقى بسيطة - هناك نزيف بسيط - قريب من النشيد الإسلامي أقرب من مقامروك

## semantic_temperature = 0.8 (t0.8)

- [ ] **Hijaz** — file: `Hijaz_20260924.wav`
  - Score: `4/5`
  - Bleed flag: ☐ no
  - Notes: لا بأس جيد - مخارج حروف صحيحة - لكن الموسيقى بسيطة - ممتاز بشكل عام للتحكم
- [ ] **Kurd** — file: `Hijaz_20260924.wav`
  - Score: `4/5`
  - Bleed flag: ☐ mild 
  - Notes: مخارج حروف صحيحة - لكن هناك نزيف بسيط تجده في الشهيق (النفس الذي يأخذه قارئ القرآن  قبل أن يبدأ بالتلاوة وهذا وجد في بيت المقدمة )
  الموسيقى أيضًا بسيطة
  أيضا لاحظ هو سكن نهاية بعض الأبيات كما يحدث في تسكين نهاية الآيات في القرآن وهو عكس ما نبحث عنه وهو الإشباع، وهذي خصيصة للورا الأساسية مقامروك.
وهنا السؤال كيف يمكننا التخفيف من قوة اللورا  للقرآن، بحيث نحصل على مخارج الحروف الصحيحة ونتنجب هذي الظواهر  التي نجدها في قراءة القرآن
مثلًا هل ألفا 0.4 أو 0.3 قد تفيد؟
## semantic_repetition_penalty = 1.4 (rp1.4)

- [ ] **Hijaz** — file: `Hijaz_20260924.wav`
  - Score: `3.5/5`
  - Bleed flag: ☐ no
  - Notes: أداء متواضع - أخطاء نجدها في الموديل الأساسي
- [ ] **Kurd** — file: `Kurd_20260924.wav`
  - Score: `3/5`
  - Bleed flag: ☐ mild 
  - Notes: عذب لفظها بالزاي وهذا خطأ نجده في الموديل الأساسي من دون لورا برون

---

## Decision rule (apply after all 8 are scored)

Per `PLAN_generation_knobs.md`'s "Verdict protocol": decide first whether a
knob moves the needle **at all**, before picking a best value.

- A knob is a **candidate** only if it shows a **clear, replicated
  direction** — i.e. it moves Hijaz and Kurd the same way (both up, both
  down, or both toward/away from bleed) relative to the `c3050_a0.5`
  reference.
- A knob with mixed or flat results across the two maqams → **no effect
  found**, drop it.
- Any candidate knob gets logged as an **open thread** in
  `claude_context.md`, not shipped into production off this single probe —
  n=2 per knob is a screening pass, not a validation.
- A knob that raises the score but introduces or worsens a bleed flag is a
  **disqualified candidate**, same rule as Step 1 (score gains that trade
  for recitation-y delivery are a failure, not a partial win).
