# 0001. Creative: user-generated content and learner data recorded from day one

- Status: Accepted
- Date: 2021-2022 (learner data saved from 09/2021, custom question engine in production by 05/2022)
- Deciders: Melchior Vila (CTO)

## Context

Br'eye teaches braille with games. Every learner is different: a teacher of a 7-year-old, a teacher of an adult who lost their sight late, and a transcriber all need different content. A fixed set of built-in games could never cover that, and we could not write content for every class ourselves.

We also knew that teachers, schools and funders would one day ask "is the learner progressing?", even though no screen needed that data yet.

## Decision

1. **Content is data, not code.** A lesson, quiz or game ("Creative") is a JSON description (blocks, questions, multiple answers, numbers, braille words) plus its audio files. The JSON is the contract: our online editor is one way to produce it, but any tool that writes valid JSON could. It is stored on the server and synced to each device over the existing WebSocket connection (quiz list check, `sync:quizzes`, update and remove handlers).
2. **Generic engines play any Creative content** (custom question factory: MCQ, multi-answer, number, block, pool, lock), so a new quiz needs no release.
3. **The same content runs on every product.** Br'eye devices and the Windows application Marble receive the same JSON and the same audio (WAV, later Opus) and interpret it with the same code. The API only says how each element is presented: displayed, written in braille, or spoken.
4. **Record learner activity from the start**, even before using it: every session (start/end, activity), every token answered, every word typed against the expected word, every block answer, with a success/error scale. Data is saved locally first (SQLite) and sent to the server when the device is online.
5. **Usage data only, no sensitive data.** Learners have no account. Only teachers log in, with their email. We store no address, age, phone number or location: only what the learner did in each exercise. Passwords are never stored in plain text.

## Alternatives considered

- Built-in games only, with content shipped in releases: simpler, but every new lesson would have meant a release and our time.
- Separate content formats per product (device, Windows app): each product could evolve freely, but every lesson would have had to be authored and tested twice.
- Record nothing until a reporting feature is designed: less storage and less work at first, but the history would start empty the day it is needed.
- Learner accounts with a personal profile: richer reports, but personal data about minors to protect, and a sign-up step teachers and children do not need.

## Consequences

- Positive: teachers adapt content to each learner without us, and new content needs no release.
- Positive: one lesson works on Br'eye and on Marble with no extra work.
- Positive: because the data was recorded from the start, a progress view per learner (braille, mistakes, progress per exercise) became a simple query when it was needed, with history going back years.
- Positive: privacy by design. With no sensitive data, compliance is simple and a data leak would expose nothing about who a learner is.
- Negative: user content needs validation and versioning (quiz ids, update/remove events, hashes for paid content); recorded data needs care (storage growth, impossible dates from devices with a wrong clock).
- Follow-up: access control for paid content packs (see 0004).

## Validation

- Unit tests for the question engines and continuity groups: `Games/CustomQuestions/Tests/custom_questions_games_test.py`, `Games/CustomQuestions/Blocks/Utils/custom_formula_evaluation_test.py`, `custom_variable_test.py`, `Games/Dependent/CustomQuestions/custom_lock_test.py`, `Source/Applications/Managers/CustomQuizz/continuity_group_manager_test.py`.
- Corrupted progression is detected and handled (`continuity_group_manager.py`, `_progressionCorrupted`).

## References

- `Games/CustomQuestions/custom_questions_factory.py` and `Games/CustomQuestions/Blocks/` (question engines)
- `Games/Dependent/CustomQuestions/` (content packs: Creative, Enigma, Lessons)
- `Source/Database/save_data.py` (`initiateSession`, `saveResultToken`, `saveResultWord`, `saveResultBlock`)
- `Source/Database/Web/Websocket/ws_events.py` (quiz sync over WebSocket)
- `Source/FilesHandling/CreativeGroup/` (groups of Creative content)
- History: learner data saving (09/2021) and export (10/2021); custom question engine (by 05/2022); per-language quizzes and permanent Creative content (06/2023); continuity groups and pools (2025); per-quiz access keys and hashes (03-04/2026)
