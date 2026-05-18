# Flaky Test Report

**Name:** Amir Shahkarami, Seyed Shayan

## Flaky Test 1

**Test name:** `de.seuhd.worldcup.WorldCupTest #evaluate returns zero when no bets are placed`

**Root cause:**
`BettingService` kept a `cachedResult` from earlier calls to `evaluate`.
`@BeforeTest` called `BettingService.clear()`, but `clear()` only removed stored bets and did
not clear the cached result. If another test evaluated non-empty bets first, this test could
receive the stale cached result even though the bet map had been cleared.

**Fix:**
I removed the cached result from `BettingService.evaluate`. The evaluation result depends on
both the current bets and the supplied match list, so caching it globally was invalid shared
state. Each call now computes the result from the current inputs.

## Flaky Test 2

**Test name:** `de.seuhd.worldcup.WorldCupTest #standings are stable when multiple teams tie on all criteria`

**Root cause:**
`StandingsService` stored accumulators in an `IdentityHashMap`. When multiple teams tied on
all explicit sorting criteria, the final order came from the map iteration order, which is
not deterministic and is unrelated to the group order.

**Fix:**
I replaced the identity map with a `LinkedHashMap` and build the final table by iterating over
`group.teams`. The existing comparator still sorts by points, goal difference, and goals for;
when all of those are equal, Kotlin's stable sort preserves the deterministic group order.

## Flaky Test 3

**Test name:** `de.seuhd.worldcup.WorldCupTest #load json from network`

**Root cause:**
The test used the real network and a 300 ms timeout. `JsonLoader.loadJsonFromNetwork()` also
shuffled a list containing working URLs and an unroutable test address. Depending on network
latency and shuffled URL order, the test could pass quickly or time out.

**Fix:**
I changed the test to use the existing injectable `UrlFetcher` parameter and return the
bundled JSON resource from the classpath. The test now verifies the loader's parsing behavior
without depending on external network timing, URL availability, or shuffled endpoint order.

## Flaky Test 4

**Test name:** `de.seuhd.worldcup.FileBettingServiceTest #test file betting with threads`

**Root cause:**
`FileBettingService.placeBet` performed a read-modify-write cycle on the same file without
synchronization. Two threads could read the same old file content, update different bets, and
then overwrite each other, so some placed bets were lost intermittently.

**Fix:**
I synchronized `placeBet` and `getBets` on the service instance. This makes the complete
read-modify-write operation atomic for one `FileBettingService`, so concurrent calls cannot
interleave and lose updates.

## Flaky Test 5

**Test name:** `de.seuhd.worldcup.FileBettingServiceTest #fresh service has no bets`

**Root cause:**
`FileBettingServiceTest` intentionally uses random test method order. The `fresh service has
no bets` test read the same shared temp file as `save bets to the shared file`. If the save
test ran first, the supposedly fresh service saw leftover bets from the earlier test.

**Fix:**
I added a `@BeforeEach` method that deletes the shared bet file before every test in the
class. This isolates each test from filesystem state left by other tests while preserving the
random method order annotation.
