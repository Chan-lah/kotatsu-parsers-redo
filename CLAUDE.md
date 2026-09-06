# CLAUDE.md — kotatsu-parsers-redo (personal fork)

Manga source parsers for the Futon app in the sibling clone `../app`.
Fork of `Kotatsu-Redo/kotatsu-parsers-redo`. Kotlin/JVM library, consumed by the app via **JitPack**
as `com.github.clquwu:kotatsu-parsers-redo:<10-char-sha>`.

**This is where source breakage gets fixed.** ~20x more churn than the app repo: 1,328 site parsers,
and roughly 136 commits here per 7 in the app over a comparable window.

Branch: work on **`perfect`**. `upstream` = Kotatsu-Redo/kotatsu-parsers-redo.

---

## 🔴 There is no CI here

`.github/workflows/` contains **only `discord.yml`**. No build check, no test run.
Upstream's original `test-parsers.yml` only ever ran `compileKotlin` — a compile check, never a
live-site test — and this fork deleted even that while keeping the test harness.

**Consequence:** source breakage is detected only by users filing issues. Nothing here verifies
that a parser still works against the live site. Building that check for the ~15-20 sources actually
read is a high-value, low-cost project (see the plan's backlog item 3).

---

## Testing a parser

The harness exists and works — it just isn't automated.

`src/test/kotlin/org/koitharu/kotatsu/parsers/MangaSources.kt`:

```kotlin
@EnumSource(MangaParserSource::class, names = [], mode = EnumSource.Mode.INCLUDE)
internal annotation class MangaSources
```

`names = []` with `Mode.INCLUDE` means **zero parsers are tested by default**. To test one, add its
`MangaParserSource` enum name to `names`, then:

```
gradlew :test --tests "org.koitharu.kotatsu.parsers.MangaParserTest"
```

Revert `names` to `[]` before committing — never commit a scoped `MangaSources`.

These are **live-site tests**: they hit the real website, so they are inherently flaky and
rate-limited. Keep the set small. JUnit 5 (`org.junit.jupiter`), not JUnit 4.

Supporting harness: `MangaLoaderContextMock.kt`, `CloudFlareInterceptor.kt`, `InMemoryCookieJar.kt`.

---

## Writing a parser

- Annotate with `@MangaSourceParser` (internal name, title, language)
- Extend `MangaParser`, or `PagedMangaParser` / `SinglePageMangaParser`
- Exactly **one** primary constructor parameter, of type `MangaLoaderContext`
- Path: `src/main/kotlin/org/koitharu/kotatsu/parsers/site/<country_code>/<Name>.kt`
  (`site/` also holds engine-family dirs — `heancms`, `foolslide`, `madara`-likes — not just country codes)
- **Never hardcode a domain.** Declare `configKeyDomain`, read the live value via `domain`.
  Domain migration is one of the most common breakage causes.
- IDs via `generateUid`; `availableSortOrders` must be non-empty
- Prefer this repo's `util` extensions over raw JSoup
- Implement `getPageUrl` when direct image links aren't available
- Optionally implement `Interceptor` for network manipulation

Common breakage causes, in order: domain migrations, HTML restructures, Cloudflare.

---

## Publishing a change back to the app

1. Commit and push to your `perfect` branch → JitPack builds on demand
2. `git rev-parse HEAD | cut -c1-10`
3. In `../app`, set `parsers = "<sha>"` in `gradle/libs.versions.toml`
4. Rebuild the app and verify in-app

JitPack builds **on demand**, so the first request for a new SHA triggers a remote build that can be
slow or time out. Retry before debugging locally. `app/libs/` in the app repo is the vendoring escape
hatch, and the app build accepts `-DparsersVersionOverride=<sha>`.

---

## Note on free-riding

Parser repair here is genuinely crowdsourced (~15 recurring contributors upstream). For most broken
sources the correct action is **bump the pin to take upstream's fix**, not write your own. Only write
a parser fix when upstream hasn't got to it.

---

## Removed from this fork

`.claude/settings.local.json` was committed upstream by accident (leaked local dev config). It
pre-granted `Bash(node test_comix_scraping.js:*)` for a script that does not exist in the repo.
Deleted on `perfect`. Do not restore it.
