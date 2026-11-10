# 0002. Multi-language by design: no hard-coded user-facing strings

- Status: Accepted
- Date: approx. December 2021 (macro keys and CSV tables), tooling extended 2024-2026
- Deciders: Melchior Vila (CTO); implemented with the team

## Context

Br'eye started in French only. Braille learning is needed in every country, and accessibility customers are often public bodies abroad (Spain, United States, Brazil, Poland…). For a blind user, every piece of text is also **spoken**, so a language is not only text: it is text, recorded or synthesised audio, keyboard layouts and braille tables.

Retrofitting translation into code full of literal strings is slow and error-prone; we wanted adding a language to be a content task, not a code change.

## Decision

1. **No user-facing string in the code.** Code refers to text through enum keys only: `Macro` (spoken output, about 1,200 keys) and `Display` (braille/screen output, about 550 keys).
2. **Text and audio file names live in CSV tables.** The main tables hold French and English columns (`text_xx`, `file_xx`, plus `_m` variants for Marble, where the wording differs on a computer: "write" instead of "place", "letter" instead of "domino"…); every other language has its own file per table. A language can fall back to a parent language (for example Belgian French to French).
3. **Grammar is data, not code.** Each counted unit (try, turn, letter, pair…) declares its singular and plural keys and the languages where it is feminine or neuter. The code asks for "3 tries" or "1 letter" and gets "une lettre", "ein Wort" or "two letters" in the right language. Languages that do not hear the plural (in French, "deux tour" and "deux tours" sound the same) need no extra recordings.
4. **Machine translation first, humans for review.** Scripts send missing cells to DeepL with a per-language glossary of terms and notes ("character" means a letter, not a person; "tú", never "usted", in Spanish…). Another script exports one language at a time for human translators, with the previous translation, the word or sentence, its context, and the base language (English or French). Every translation records who produced it: DeepL, reviewed by the team, a professional translator, or a braille teacher, since braille translation needs a dedicated specialist.
5. **Audio generated, not recorded by hand:** Azure Text-to-Speech generates the audio from the tables, after a confirmation step that shows what will be generated.
6. **Audio per language in its own git submodule** (WAV files at first, Opus now), so a device downloads only the languages it uses. Language-specific content (dictionaries, word lists, crosswords) lives with the games' dependencies.
7. A documented playbook for adding a language (`creating_a_new_lang.md`).

## Alternatives considered

- gettext/`.po` files: standard for text, but does not carry the audio file per string that a screen-reader-first product needs.
- Project Fluent or ICU MessageFormat (with CLDR plural rules): handle plural and gender well, but they format text strings at runtime. Our output is a sequence of pre-recorded audio clips, so the grammar has to choose keys, not format strings.
- Translating strings inline when needed: fastest at first, but each new language would touch the whole codebase.
- Live text-to-speech instead of audio files: no files to manage, but it needs a network connection or a heavy on-device engine. Offline engines sound robotic, which is a problem for children learning to read.
- A SQLite database instead of CSV: easier to query, but CSV diffs are readable in pull requests and translators open them in a spreadsheet.
- A translation management platform (Crowdin, Weblate…): built-in review workflow, but no audio file per string, no braille-teacher review step, a recurring cost, and content stored outside our systems.
- Machine translation only: fast, but not acceptable for a teaching product, especially for braille.
- All languages bundled in the application: simpler to ship, but every device would download every language.

## Consequences

- Positive: the products went from French only to 11 languages. Adding a language is mostly translation, review and audio generation.
- Positive: fewer bugs from typos in user text; text changes do not need a code review of logic.
- Positive: the source of each translation is known, so review effort goes where it is needed.
- Negative: indirection (a key to look up for every sentence), large CSV files, and a build step to regenerate audio.
- Negative: each new language must fill in the grammar table (gender and plural) for every unit.
- Follow-up: language changes at runtime are propagated through the Observer (see 0003).

## Validation

- Unit tests check that every `Macro` and `Display` key has its text and audio file in every language (no missing file). GitHub Actions runs them on every pull request, before the merge to master.
- Unit tests exercise the games through the key tables.
- Check scripts verify, before release, that the text, audio files and dictionaries exist and are used in each language.

## References

- `Source/OutputHandling/macro_enum.py`, `Source/OutputHandling/display_enum.py` (Macro / Display enums), `Source/Utils/Leaves/language.py` (languages and fallback), `Source/Utils/Leaves/score_utils.py` (units, plural and gender), `Resources/CSV/`, `Resources/CSV/Lang/`
- `Resources/Tools/CleanScripts/translate_csv.py` and `glossary_xx.csv` (DeepL and glossaries), `Resources/Tools/CSV_Tools/for_translation.py` (export for human translators), `Resources/Tools/Generation/` and `Resources/Tools/Audios/` (Azure TTS), `Resources/Tools/Checks/`, `.github/workflows/python-app.yml`
- `Resources/Documentation/creating_a_new_lang.md`, `Resources/Documentation/audio_submodules_scripts.md`
- History: "csv" (2021-12-22), "changes macro names" (2021-12-23), English tutorial (03/2023, completed 05/2025), "feat: deepl translation" (2025-04-01), "feat(git submodule): changes" (2025-10-15), "deepl now works?" (2025-10-16), Brazilian Portuguese added with the new-language guide (10/2025), Arabic pack skeleton (2026-10-13)
