# 0007. Russian is a third bundle, and the draft recognizers follow it

Status: Accepted (2026-10-04)

## Context

The Studio shipped with two message bundles, `en` and `pt`. `pt` is complete Brazilian Portuguese, written by hand
for the pre-v1 Studio, and currently unreachable through Settings — the Hermes locale that would select it does not
exist yet in the host. `i18n-ui.js` says so: *"unused until the app ships a pt locale"*.

Adding Russian raised four questions the two-locale setup had never had to answer.

1. **What is a locale, now that there are three?** `localeOf` was a binary test — `t('locale') === 'pt' ? 'pt' : 'en'` —
   and it is the single funnel for every core question, every warning and every `locale` field sent to the backend.
   A third locale had to change its shape or it would silently degrade Russian to English everywhere at once.
2. **Does a Russian draft get understood?** The draft recognizers (`detection-core.js`) are regex-based and cover
   Portuguese and English only; every pattern carries an explicit `Languages:` comment saying so. A Russian draft
   would fall through to defaults and the Studio would pick the wrong fields for it — a Russian interface that does
   not understand Russian input is not a Russian Studio.
3. **Which layers become Russian?** `locale` reaches the backend purely as the instruction language for the model's
   human-facing notes. The prompt itself is assembled from the draft, in the draft's language. So the layers can be
   separated: the interface, the questions and the notes follow the Studio language; the prompt follows the request.
4. **What happens to `pt`?** Every new string now needs a third translation, and a bundle nobody can select is
   maintenance that grows silently.

## Decision

- **A locale is a member of a list, not a branch.** `localeOf` resolves through a membership test against the known
  locales and falls back to `en`, the complete bundle. Adding a locale is now a list edit, not a logic edit.
- **Russian is a full third bundle.** `en`, `pt` and `ru` ship side by side; `pt` stays reachable in the code and its
  saved ladders keep mapping back to their option values.
- **The draft recognizers cover Russian.** Category, artifact, verb, question and politeness patterns get Russian
  words, and the tests pin one positive per category plus the everyday-Russian negatives that must not read as a
  task. `NONE_WORDS` and `DESIGN_DEFAULT_WORDS` get their Russian labels, because those two lists are the one place
  where a localized label is matched against human-readable text by hand instead of by value — and both are
  compared against folded text, so their entries are written folded (`рекомендуемыи список`, not a transliteration).
  A Russian stem may not be followed by `\w*`: `\w` is ASCII-only, so that would match only the bare stem.
- **Layers split by audience.** Interface, questions, option labels and AI notes follow the Studio language. `guide`
  strings stay English in every bundle, as they already do in `pt`, because they are instructions for the model and
  never shown to a human. The prompt follows the request language, never the Studio language.
- **Russian is selected explicitly.** `auto` still follows Hermes, which has no `ru` locale, so Russian arrives
  through Settings. No operating-system locale fallback is added: Hermes does not expose one through the SDK.
- **The reverse map scales for free.** `answerToValue` iterates every locale in `CORE_MESSAGES`, so Russian option
  labels are readable back into their values with no new code.

## Consequences

- A Russian draft is classified as well as a Portuguese one; the recognizers are larger and every pattern now carries
  three languages, so the `Languages:` comments must stay honest or the next agent will trust a stale one.
- Every new user-visible string needs three translations, and Russian is the one that will rot first if a contributor
  only knows two of the three. The "ru differs from en" test exists to catch the exact string someone copied.
- `pt` remains unreachable through Settings. That is now a deliberate cost, not an oversight, and the ADR that
  removes it has to name this file.
- The AI notes language is the one place where the Studio language leaks into a model's output, and it follows the
  Studio language, not the request language.
- A Russian ladder saved today still parses after a future rename of a Russian label, because `answerToValue` tries
  every bundle — but a *typo* in a Russian label is not recoverable, and no test can catch it.

Revisit when Hermes ships an `ru` locale (`auto` starts working by itself), or when a fourth locale would make the
parity tests slow enough to matter.