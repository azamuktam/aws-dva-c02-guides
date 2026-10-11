# HOW TO STUDY FROM THIS
**Don’t read it front to back.** It’s a reference, and reading a reference cover-to-cover produces recognition, not recall.
**A sequence that works:**
1. **Build something first.** Deploy an API Gateway → Lambda → DynamoDB application with SAM. Add SQS, then SNS, then Step Functions. Break it on purpose: set the visibility timeout too low and watch duplicates appear; remove the log permission and watch logs vanish. Every failure mode in this document becomes memorable once you’ve caused it.
2. **Then read the sections for what you built.** The “how it works” and “how it fails” parts will now describe things you’ve seen.
3. **Then read Part 4 (Security) properly.** It’s 26%, it’s the domain where borderline candidates fail, and it’s the hardest to learn from building alone because most of it is invisible when it works.
4. **Then Part 5 (Deployment).** Wire up a real CodePipeline with a buildspec and an appspec. The hook names and phase names stop being arbitrary once you’ve written the files.
5. **Then take timed mock exams.** 65 questions, 130 minutes. Read the explanation for **every** question, including the ones you got right — a right answer for the wrong reason will not survive rephrasing.
6. **Target 80%+ consistently** on mocks before booking.
**When you get a question wrong, ask which of the five question types it was** (Part 0). If it was type 2 (“why is this broken?”), you’re missing a mechanism, and the fix is to reread the “how it works” section — not to memorise the answer.
**Two weeks out**, switch from reading to Part 8. If a row surprises you, go back to its section.
**The morning of the exam**, read only Part 8 and Part 0.
**During the exam:** answer every question, flag the ones you’re unsure of, and move on. Time pressure causes more failures than knowledge gaps. On a genuine tie, apply the three heuristics: roles over credentials, managed over custom, least privilege first.
