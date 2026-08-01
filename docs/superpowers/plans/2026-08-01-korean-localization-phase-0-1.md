# 한국어화 0–1단계 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** gen1recomp 화면에 한글이 정상적으로 뜨고, 번역자가 안전하게 작업을 시작할 수 있는 상태를 만든다.

**Architecture:** 번역문은 별도 콘텐츠 모드로 나가고, 이 저장소에는 그 모드가 동작하기 위한 엔진 수정만 들어간다. 엔진 수정은 전부 업스트림 제출 가능한 형태로, 주제별 브랜치 1개 = PR 1개. 한글 글리프는 8px 폭 × 16px 높이 셀로, `Font.lua` 에 페이지별 셀 높이 개념을 추가해 지원한다.

**Tech Stack:** Lua 5.1 / LuaJIT 2.1 (LÖVE 11.5 임베드), Python 3 + Pillow + fontTools (빌드 도구), RGBDS 1.0.2 (ROM 공급원)

**Spec:** `docs/superpowers/specs/2026-08-01-korean-localization-design.md`

---

## Global Constraints

- **영문 출력은 바이트 동일해야 한다.** 한글 폰트가 로드되지 않은 상태(=바닐라 부팅)에서 모든 렌더 결과가 변경 전과 정확히 같아야 한다. `tests/parity_text_wrap.lua`, `tests/parity_text_scroll_bounds.lua` 가 이를 지킨다.
- **인터프리터는 LuaJIT.** Lua 5.4 전용 문법 금지. `scripts/test.sh` 의 기본 `LUA=luajit`.
- **엔진 패치는 `upstream/dev` 에서 브랜치를 딴다.** `ko` 에서 따면 한국어화 커밋이 PR 에 섞인다.
  ```sh
  git fetch upstream && git checkout -b <브랜치명> upstream/dev
  ```
- **작업 완료 조건은 `scripts/test.sh` 전 티어 통과.** CI 는 ROM 이 없어 T3 를 스킵하므로 로컬에서 반드시 직접 돌린다.
- **`tests/modkit_tests.lua` 는 `tests/run_modkit.lua` 가 아니라 `tests/run_tests.lua`(T3 티어)에서 `dofile` 된다.** `run_modkit.lua` 는 `tests/modkit/cases/` 와 `mods/*/tests` 만 훑는다. 이 파일의 테스트를 돌릴 때는 `luajit tests/run_tests.lua` 를 쓴다.
- **`scripts/test.sh` 만 믿지 말고 종료코드를 직접 본다.** `run_content_behavior()` 가 `^FAIL ` 라인 수만 세고 종료코드를 무시해서, T3 가 **크래시하면 FAIL 라인이 0개라 PASS 로 보고**된다. Task 1.5 에서 고치기 전까지는 반드시 `luajit tests/run_tests.lua; echo $?` 로 직접 확인한다.
- **커밋 메시지는 영어**, 코드 주석은 주변 코드와 같은 언어(영어)로 쓴다. 이 저장소는 업스트림에 제출된다.
- 기준 리비전: `upstream/dev` @ `50947ee`.

### 테스트 하네스 사용법

`tests/harness.lua` 는 두 가지 형태를 제공한다. 새 스위트는 `T.suite(label)` 형태를 쓴다:

```lua
package.path = "./?.lua;./?/init.lua;" .. package.path
local S = require("tests.harness").suite("suite name")
local check, eq = S.check, S.eq

check(condition, "메시지")
eq(actual, expected, "메시지")

S.finish()
```

`love` 전역이 필요하면 `if not _G.love then _G.love = require("tests.love_stub") end`.

---

# Phase 0 — 도구가 번역자를 배신하지 않게

**이 단계가 끝나면:** `modkit translation` 스캐폴더가 안전해지고, 한글 이름이 어디서도 바이트 단위로 잘리지 않는다. 화면에 한글은 아직 안 나온다.

**왜 먼저인가:** 5개 버그 모두 *스캐폴드 직후에는 멀쩡하고 번역자가 작업을 시작한 뒤에* 터진다. 그중 3개는 조용히 실패한다. 번역을 시작하기 전에 고치지 않으면 며칠치 작업이 날아간다.

---

### Task 1: modkit `translation` 테스트 하네스

**왜 이것이 먼저인가:** `grep -n translation tests/modkit_tests.lua` 가 **0건**이다. 685줄짜리 테스트 파일이 이 서브커맨드를 전혀 다루지 않는다. 버그 5개가 30커밋 동안 살아남은 이유가 이것이므로, 버그를 고치기 전에 잡는 그물을 먼저 친다.

**Files:**
- Modify: `tests/modkit_tests.lua` (파일 끝에 추가)

**Interfaces:**
- Produces: `run_modkit(args)` 헬퍼 — Task 2–5 가 사용한다. 시그니처: `run_modkit(argsString) -> exitCode, stdout`

- [ ] **Step 1: 기존 테스트 파일의 서브프로세스 호출 패턴 확인**

```sh
grep -n 'io.popen\|os.execute\|python3' tests/modkit_tests.lua | head -20
```

기존 스위트가 modkit 을 어떻게 실행하는지 그대로 따른다. 새 방식을 발명하지 않는다.

- [ ] **Step 2: 실패하는 테스트를 쓴다 — 스캐폴드가 기대한 파일을 만드는가**

`tests/modkit_tests.lua` 끝에 추가. (`TMP` 와 `run_modkit` 은 Step 1 에서 확인한 기존 헬퍼 이름에 맞춘다. 없으면 아래처럼 만든다.)

```lua
-- ---------------------------------------------------------------- translation
-- The `translation` subcommand had zero coverage, which is why five defects
-- lived in it for ~30 commits.  These cases pin the scaffold's shape and the
-- refresh contract; every one of them failed before the fixes that follow.

local function tmpdir(name)
  local base = os.getenv("TMPDIR") or "/tmp"
  local path = base .. "/modkit_" .. name .. "_" .. tostring(os.time())
  os.execute("rm -rf " .. path)
  return path
end

local function run_modkit(argstr)
  local pipe = io.popen("python3 tools/modkit.py " .. argstr .. " 2>&1")
  local out = pipe:read("*a")
  local ok, _, code = pipe:close()
  return (ok and 0 or (code or 1)), out
end

local function file_exists(p)
  local f = io.open(p, "r")
  if f then f:close() return true end
  return false
end

local function read_file(p)
  local f = io.open(p, "r")
  if not f then return nil end
  local s = f:read("*a")
  f:close()
  return s
end

do
  local dest = tmpdir("scaffold")
  local code, out = run_modkit("translation kotest --language korean --dest " .. dest)
  eq(code, 0, "translation scaffold exits 0")
  local root = dest .. "/kotest"
  for _, f in ipairs({ "manifest.json", "main.lua", "README.md", "TRANSLATING.md",
                       "lang/dialogue.lua", "lang/strings.lua",
                       "lang/species_names.lua", "lang/font.lua",
                       "lang/charmap.lua", "lang/naming.lua" }) do
    check(file_exists(root .. "/" .. f), "scaffold writes " .. f)
  end
  os.execute("rm -rf " .. dest)
end
```

- [ ] **Step 3: 테스트를 돌려 통과하는지 확인**

```sh
cd /Users/hwanmooy/Dropbox/dev/pokemon/gen1recomp
luajit tests/run_tests.lua 2>&1 | tail -20
```

기대: PASS. 스캐폴드 자체는 정상 동작하므로 이 케이스는 통과해야 한다. **실패하면 헬퍼가 잘못된 것이니 Step 1 로 돌아간다.**

- [ ] **Step 4: 커밋**

```sh
git add tests/modkit_tests.lua
git commit -m "test: cover modkit translation scaffold output"
```

---

### Task 2: 버그 1 + 1b — 네이밍 훅 API 와 시그니처

**Files:**
- Modify: `tools/modkit.py:1518` (`TRANSLATION_MAIN` 안의 훅 등록)
- Test: `tests/modkit_tests.lua`

**Interfaces:**
- Consumes: Task 1 의 `run_modkit`, `tmpdir`, `read_file`

**증상:** 스캐폴더가 `mod.hooks:on(...)` 을 생성하는데 모드 API 에는 `wrap` 만 있다 (`src/mods/Loader.lua:559-561`, `src/mods/Hooks.lua`). 번역자가 `lang/naming.lua` 를 채우는 순간 `attempt to call method 'on' (a nil value)` 로 **번역 모드 전체가 롤백**된다. `on` 을 `wrap` 으로 바꿔도 시그니처가 다르다 — wrap 콜백은 `callback(nextFn, unpack(args))` 로 호출된다 (`src/mods/Hooks.lua:66`).

- [ ] **Step 1: 실패하는 테스트를 쓴다**

`tests/modkit_tests.lua` 의 translation 블록에 추가:

```lua
do
  local dest = tmpdir("hook")
  run_modkit("translation kohook --language korean --dest " .. dest)
  local main = read_file(dest .. "/kohook/main.lua")
  check(main ~= nil, "hook case: main.lua readable")
  check(not main:find("hooks:on", 1, true),
    "generated main.lua must not call hooks:on -- the mod API only exposes wrap")
  check(main:find("hooks:wrap", 1, true) ~= nil,
    "generated main.lua registers the naming grid through hooks:wrap")
  -- wrap callbacks are invoked as callback(next, ...), so the generated
  -- closure has to take `next` first or every argument shifts by one
  check(main:find("function%(next, base, ctx%)") ~= nil,
    "generated wrap callback takes (next, base, ctx)")
  os.execute("rm -rf " .. dest)
end
```

- [ ] **Step 2: 테스트를 돌려 실패를 확인**

```sh
luajit tests/run_tests.lua 2>&1 | grep -E 'hooks:on|hooks:wrap|wrap callback'
```

기대: 3건 FAIL.

- [ ] **Step 3: 최소 구현**

`tools/modkit.py:1518` 부근을 아래로 교체:

```python
    -- hooks are wrap-style: the callback's first argument is next(), and
    -- calling it with no arguments runs the rest of the chain (ending in
    -- the engine's own grid) with the arguments this link received.
    mod.hooks:wrap("ui.naming.grid", function(next, base, ctx)
      local want = ctx.lower and grid.lower or grid.upper
      return want or next()
    end)
```

(원본은 `mod.hooks:on("ui.naming.grid", function(base, ctx)` … `return want or base`.)

- [ ] **Step 4: 테스트 통과 확인**

```sh
luajit tests/run_tests.lua 2>&1 | tail -5
```

기대: ALL TESTS PASSED.

- [ ] **Step 5: 커밋**

```sh
git add tools/modkit.py tests/modkit_tests.lua
git commit -m "fix(modkit): translation stub used a nonexistent hooks:on

The mod API exposes only hooks:wrap (Loader.lua:559-561), and a wrap
callback is invoked as callback(next, ...) (Hooks.lua:66). The generated
stub called :on with an (base, ctx) signature, so filling in lang/naming.lua
threw and the loader rolled the entire translation back."
```

---

### Task 3: 버그 2 — 폰트 경로가 모드 밖을 가리킨다

**Files:**
- Modify: `tools/modkit.py:1478` 부근 (`TRANSLATION_MAIN` 의 폰트 등록 루프)
- Test: `tests/modkit_tests.lua`

**증상:** 스텁이 `image = "assets/font/<id>.png"` 를 쓰는데, `src/render/Assets.lua:37` 의 `resolve()` 는 `assets/generated/` 접두어가 아니면 그대로 반환한다. 모드 파일은 `mods/<id>/` 아래 있으므로 게임 루트에서 찾다가 실패하고, `src/render/Font.lua:43` 의 `pcall` 이 조용히 삼킨다. **글자가 안 나오는데 에러도 로그도 없다.**

- [ ] **Step 1: 실패하는 테스트를 쓴다**

```lua
do
  local dest = tmpdir("fontpath")
  run_modkit("translation kofont --language korean --dest " .. dest)
  local main = read_file(dest .. "/kofont/main.lua")
  -- a bare assets/... path is looked up against the game root, and Font.load
  -- pcalls the miss with no log, so the translator gets blank text and no clue
  check(main:find("assets:path", 1, true) ~= nil,
    "font page image path is made mod-absolute with mod.assets:path()")
  os.execute("rm -rf " .. dest)
end
```

- [ ] **Step 2: 테스트를 돌려 실패를 확인**

```sh
luajit tests/run_tests.lua 2>&1 | grep 'assets:path'
```

기대: FAIL.

- [ ] **Step 3: 최소 구현**

`tools/modkit.py` 의 `for id, page in pairs(catalog("font")) do` 루프를 아래로 교체:

```python
  -- A page's `image` is written relative to this mod, but the engine's
  -- asset search only rewrites assets/generated/*, so a bare "assets/..."
  -- would be looked up against the game root and Font.load would pcall the
  -- miss and silently drop the page -- no glyphs, no error.  Making it
  -- absolute with mod.assets:path() is what puts it inside the mod.
  for id, page in pairs(catalog("font")) do
    if type(page) == "table" and type(page.image) == "string" then
      local resolved = {}
      for k, v in pairs(page) do resolved[k] = v end
      resolved.image = mod.assets:path(page.image)
      page = resolved
    end
    mod.content.font:register(id, page)
  end
```

- [ ] **Step 4: 테스트 통과 확인**

```sh
luajit tests/run_tests.lua 2>&1 | tail -5
```

- [ ] **Step 5: 커밋**

```sh
git add tools/modkit.py tests/modkit_tests.lua
git commit -m "fix(modkit): translation font page path never resolved into the mod

Assets.resolve only rewrites assets/generated/*, so the stub's bare
assets/font/<id>.png was looked up against the game root; Font.load pcalls
the failure with no else branch, so a translator shipping a font sheet got
blank text and zero diagnostics."
```

---

### Task 4: 버그 3 — `--refresh` 가 보류 번역을 삭제한다

**Files:**
- Modify: `tools/modkit.py:1884` (`_read_existing_catalog` 의 정규식)
- Test: `tests/modkit_tests.lua`

**증상:** `_merge_catalog` 는 orphan 을 `--   ["KEY"] = "값",` 주석으로 park 하는데, `_read_existing_catalog` 의 정규식이 `^\s*\[` 라 `--` 를 못 넘는다. 두 번째 `--refresh` 에서 그 줄들이 안 읽히므로 **보류해둔 번역이 조용히 사라진다.**

- [ ] **Step 1: 실패하는 테스트를 쓴다**

```lua
do
  local dest = tmpdir("orphan")
  run_modkit("translation koorph --language korean --dest " .. dest)
  local cat = dest .. "/koorph/lang/strings.lua"
  -- park a translation under a key the harvest will not produce
  local body = read_file(cat)
  body = body:gsub("^return {", 'return {\n  ["__PARKED_TEST_KEY__"] = "보류된 번역",', 1)
  local f = io.open(cat, "w"); f:write(body); f:close()

  run_modkit("translation koorph --language korean --dest " .. dest .. " --refresh")
  local after1 = read_file(cat)
  check(after1:find("보류된 번역", 1, true) ~= nil,
    "refresh #1 parks the orphaned translation rather than dropping it")

  run_modkit("translation koorph --language korean --dest " .. dest .. " --refresh")
  local after2 = read_file(cat)
  check(after2:find("보류된 번역", 1, true) ~= nil,
    "refresh #2 still has it -- a parked row must survive repeated refreshes")
  os.execute("rm -rf " .. dest)
end
```

- [ ] **Step 2: 테스트를 돌려 실패를 확인**

```sh
luajit tests/run_tests.lua 2>&1 | grep 'refresh #'
```

기대: `refresh #1` PASS, `refresh #2` FAIL.

- [ ] **Step 3: 최소 구현**

`tools/modkit.py` 의 `_read_existing_catalog` 안:

```python
    # The optional `--` is what keeps a parked translation alive across a
    # second --refresh: _merge_catalog writes orphans as commented rows, and
    # a reader that only saw live rows would drop them on the next run.
    entry = re.compile(r'^\s*(?:--\s*)?\[(.+?)\]\s*=\s*("(?:[^"\\]|\\.)*")\s*,')
```

- [ ] **Step 4: 테스트 통과 확인**

```sh
luajit tests/run_tests.lua 2>&1 | tail -5
```

- [ ] **Step 5: 커밋**

```sh
git add tools/modkit.py tests/modkit_tests.lua
git commit -m "fix(modkit): second --refresh silently dropped parked translations

_merge_catalog parks orphans as commented rows but _read_existing_catalog's
regex could not cross the leading --, so the next refresh could not see them
and wrote them out of existence."
```

---

### Task 5: 버그 4 — `TRANSLATING.md` 가 번역자 메모를 지운다

**Files:**
- Modify: `tools/modkit.py:1784-1785`
- Test: `tests/modkit_tests.lua`

**증상:** `emit("TRANSLATING.md", TRANSLATING_MD)` 가 `overwrite` 기본값 `True` 로 호출된다. 이웃한 `manifest.json`/`main.lua`/`README.md` 는 전부 `overwrite=not exists` 인데 이 두 줄만 다르다. 같은 함수의 주석이 "A refresh must never clobber hand-edited prose" 라고 적혀 있으므로 명백한 실수다.

- [ ] **Step 1: 실패하는 테스트를 쓴다**

```lua
do
  local dest = tmpdir("clobber")
  run_modkit("translation koclob --language korean --dest " .. dest)
  local doc = dest .. "/koclob/TRANSLATING.md"
  local f = io.open(doc, "a"); f:write("\n## 번역자 메모: 지우지 말 것\n"); f:close()

  run_modkit("translation koclob --language korean --dest " .. dest .. " --refresh")
  check(read_file(doc):find("번역자 메모", 1, true) ~= nil,
    "refresh must not clobber hand-edited TRANSLATING.md")

  local fdoc = dest .. "/koclob/assets/font/README.md"
  local g = io.open(fdoc, "a"); g:write("\n## font note\n"); g:close()
  run_modkit("translation koclob --language korean --dest " .. dest .. " --refresh")
  check(read_file(fdoc):find("font note", 1, true) ~= nil,
    "refresh must not clobber hand-edited assets/font/README.md")
  os.execute("rm -rf " .. dest)
end
```

- [ ] **Step 2: 테스트를 돌려 실패를 확인**

```sh
luajit tests/run_tests.lua 2>&1 | grep clobber
```

기대: 2건 FAIL.

- [ ] **Step 3: 최소 구현**

```python
    emit("TRANSLATING.md", TRANSLATING_MD, overwrite=not exists)
    emit("assets/font/README.md", FONT_README, overwrite=not exists)
```

- [ ] **Step 4: 테스트 통과 확인 + 전체 회귀**

```sh
luajit tests/run_tests.lua 2>&1 | tail -5
./scripts/test.sh 2>&1 | tail -5
```

기대: 둘 다 통과.

- [ ] **Step 5: 커밋 + PR 브랜치 푸시**

```sh
git add tools/modkit.py tests/modkit_tests.lua
git commit -m "fix(modkit): refresh overwrote hand-edited TRANSLATING.md

Its neighbours all pass overwrite=not exists; these two lines did not,
against the stated intent of the surrounding comment."
git push -u origin HEAD
```

> 이 시점에서 Task 1–5 를 묶은 브랜치가 **업스트림 PR #1** 이 된다. 제목 예: `fix(modkit): five defects in the translation scaffolder, plus the tests that were missing`

---

### Task 6: 이름·표시 문자열을 글리프 단위로 자르기

**Files:**
- Modify: `src/render/Font.lua` (`Font.glyphCount`, `Font.cut` 추가)
- Modify: `src/link/Protocol.lua:145`
- Modify: `src/link/Handshake.lua:173`, `src/link/LinkState.lua:728,736`
- Modify: `src/mods/ManagerState.lua:52,961`, `src/ui/QuarantineReport.lua:18`, `src/ui/TownMap.lua:412`, `src/dev/Console.lua:344`
- Modify: `tools/save-editor/Theme.lua:207`, `tools/save-editor/Kit.lua:357`
- Test: `tests/parity_glyph_safe_names.lua` (신규)

**심각도 구분:** `src/link/Protocol.lua:145` 만이 **잘린 값을 세이브에 기록**한다. 나머지는 표시 전용이다. 세이브 에디터(`tools/save-editor/`)는 `src.render.Font` 를 쓰지 않는 별도 LÖVE 앱이므로 자체 UTF-8 순회 헬퍼로 고친다.

- [ ] **Step 1: 실패하는 테스트를 쓴다**

`tests/parity_glyph_safe_names.lua` 를 새로 만든다:

```lua
-- Name caps are GLYPH counts, not byte counts.  The naming screen counts
-- typed cells (NamingScreen.glyphs), so a cap of 7 means 7 glyphs; `#name`
-- only agrees for ASCII.  A Hangul syllable is 3 UTF-8 bytes, so a byte cut
-- halves it and leaves an unrenderable fragment -- and the link protocol
-- files that fragment into the receiver's save.
package.path = "./?.lua;./?/init.lua;" .. package.path
if not _G.love then _G.love = require("tests.love_stub") end

local S = require("tests.harness").suite("glyph-safe names")
local check, eq = S.check, S.eq

local Font = require("src.render.Font")

local KO = "김철수영희민준"     -- 7 syllables, 21 bytes
eq(Font.glyphCount(KO), 7, "glyphCount counts syllables, not bytes")
eq(Font.glyphCount("ABCDEFG"), 7, "glyphCount matches # for ASCII")
eq(Font.glyphCount(""), 0, "glyphCount of empty string is 0")
eq(Font.glyphCount(nil), 0, "glyphCount of nil is 0")

eq(Font.cut(KO, 3), "김철수", "cut keeps whole syllables")
eq(Font.cut(KO, 7), KO, "cut at exactly the length is a no-op")
eq(Font.cut(KO, 99), KO, "cut past the end is a no-op")
eq(Font.cut(KO, 0), "", "cut to zero is empty")
eq(Font.cut("ABCDEFG", 3), "ABC", "cut is byte-identical for ASCII")

-- every prefix must be valid UTF-8: no dangling continuation bytes
for n = 0, 7 do
  local out = Font.cut(KO, n)
  eq(#out % 3, 0, ("cut(%d) lands on a syllable boundary"):format(n))
end

-- the severe case: the link protocol persists what it truncates
local Protocol = require("src.link.Protocol")
local data = { pokemon = { PIKACHU = { name = "PIKACHU", baseStats = {} } } }
local mon = Protocol.unpackMon(data, {
  species = "PIKACHU", level = 5, ot = KO, nickname = KO,
  moves = {}, stats = {}, hp = 20,
})
eq(mon.ot, Font.cut(KO, 10), "received OT is cut on a glyph boundary")
check(mon.nickname == nil or Font.glyphCount(mon.nickname) <= 10,
  "received nickname is capped in glyphs")

S.finish()
```

- [ ] **Step 2: 테스트를 돌려 실패를 확인**

```sh
luajit tests/parity_glyph_safe_names.lua
```

기대: `Font.glyphCount` 가 없어 에러. 그다음 `Font.cut` 도 없음.

- [ ] **Step 3: `Font.glyphCount` / `Font.cut` 구현**

`src/render/Font.lua` 의 `Font.spansFitting` 바로 뒤에 추가:

```lua
-- Glyph count of a string.  This is what every *name length cap* means:
-- the naming screen counts cells typed (NamingScreen.glyphs), so a cap of
-- 7 is 7 glyphs, and `#name` is only the same number for ASCII.  A Hangul
-- syllable is 3 UTF-8 bytes, so `#name` triples every Korean name.
function Font.glyphCount(text)
  return #Font.split(tostring(text or ""))
end

-- Truncate to at most `maxGlyphs` whole glyphs.  Never lands inside a
-- charmap sequence or a UTF-8 codepoint, so it cannot halve a 3-byte
-- Hangul syllable the way `text:sub(1, n)` does.
function Font.cut(text, maxGlyphs)
  text = tostring(text or "")
  if maxGlyphs <= 0 then return "" end
  local spans = Font.split(text)
  if #spans <= maxGlyphs then return text end
  return text:sub(1, spans[maxGlyphs].to)
end
```

- [ ] **Step 4: 링크 프로토콜 수정 (심각 — 세이브에 기록됨)**

`src/link/Protocol.lua` 의 `unpackMon` 안:

```lua
  -- Cut in GLYPHS, not bytes: the naming screen's caps are glyph counts
  -- (NamingScreen counts typed cells), and `:sub(1, 10)` on a Korean OT
  -- ("김철수" is 9 bytes) slices a 3-byte Hangul syllable in half and
  -- files the broken bytes into the receiver's save.
  local Font = require("src.render.Font")
  local ot = type(packed.ot) == "string" and Font.cut(packed.ot, 10) or nil
  local nickname = type(packed.nickname) == "string"
    and Font.cut(packed.nickname, 10) or nil
```

그리고 반환 테이블의 `nickname = packed.nickname` 을 `nickname = nickname` 으로 바꾼다.

- [ ] **Step 5: 테스트 통과 확인**

```sh
luajit tests/parity_glyph_safe_names.lua
```

기대: 전부 PASS.

- [ ] **Step 6: 표시 전용 사이트들을 같은 헬퍼로 교체**

참조 패치가 `/private/tmp/claude-501/.../scratchpad/glyph-safe-names.clean.patch` 에 있고 `git apply --check` 를 통과한다. 각 사이트는 다음 형태다:

| 파일 | 변경 |
|---|---|
| `src/link/Handshake.lua:173` | `#text > WIDTH` → `Font.glyphCount(text) > WIDTH`, 하드컷은 `Font.cut` |
| `src/link/LinkState.lua:728,736` | `(mon.nickname or def.name):sub(1, 8)` → `Font.cut(..., 8)` |
| `src/mods/ManagerState.lua:52` | `#word > width` → `Font.glyphCount(word) > width` |
| `src/mods/ManagerState.lua:961` | `drawTruncated` 를 `Font.draw(Font.cut(text, cols), x, y)` 로 |
| `src/ui/QuarantineReport.lua:18` | `clip` 을 `Font.cut(text, WIDTH)` 로 |
| `src/ui/TownMap.lua:412` | `#loc.name * 8` → `Font.width(loc.name)` |
| `src/dev/Console.lua:344` | 백스페이스를 `Font.cut(buf, Font.glyphCount(buf) - 1)` 로 |

`src/link/Handshake.lua` 는 `local Font = require("src.render.Font")` 를 상단에 추가해야 한다.

- [ ] **Step 7: 세이브 에디터의 바이트 순회 수정**

`tools/save-editor/` 는 `src.render.Font` 를 쓰지 않는 LÖVE 네이티브 UI 다. `Theme.lua` 에 자체 헬퍼를 추가한다:

```lua
-- Iterate whole UTF-8 characters.  Per-BYTE iteration printed each byte of
-- a Hangul syllable as its own (unrenderable) glyph and measured it 3x too
-- wide; every text helper below walks characters instead.
local function chars(text)
  local out, i, n = {}, 1, #text
  while i <= n do
    local b = text:byte(i)
    local len = (b < 0x80 and 1) or (b < 0xE0 and 2) or (b < 0xF0 and 3) or 4
    out[#out + 1] = text:sub(i, i + len - 1)
    i = i + len
  end
  return out
end
Theme.chars = chars
```

그리고 `Theme.spaced`, `Theme.spacedWidth`, `Theme.ellipsize`, `Theme.ellipsizeLeft` 의 `for i = 1, #text` 바이트 루프를 `for _, ch in ipairs(chars(...))` 로 바꾼다. `Kit.lua:357` 의 백스페이스는 `Theme.chars` 로 마지막 문자를 제거한다.

- [ ] **Step 8: 전체 회귀 + 커밋**

```sh
./scripts/test.sh 2>&1 | tail -5
```

기대: ALL TIERS PASSED. **여기서 실패하면 영문 동작이 바뀐 것이므로 되돌린다.**

```sh
git add -A src/ tools/save-editor/ tests/parity_glyph_safe_names.lua
git commit -m "fix: cut names by glyphs, not bytes

Every name cap in the engine is a glyph count -- NamingScreen counts typed
cells -- but several sites clamped with :sub(). A 3-byte Hangul syllable
gets halved, and src/link/Protocol.lua:145 files the broken bytes into the
receiving save. Adds Font.glyphCount/Font.cut and routes the sites through
them; the save editor, which does not use src.render.Font, gets its own
UTF-8 character iterator."
git push -u origin HEAD
```

> **업스트림 PR #2.** 한국어와 무관하게 일본어·중국어·유럽어 악센트 전부에 해당하는 버그다.

---

### Task 7: 커버리지 게이트를 조인다

**왜 지금인가:** 미래핑 문자열 390개를 고치기 전에 게이트부터 조여야 한다. 안 그러면 고치는 동안 새 미래핑이 계속 들어온다.

**Files:**
- Modify: `tests/engine/gate_strings_coverage.lua`

**현재 구멍 3개:**
1. `offenders()` 가 `\n`·`\f`·`\v` 를 포함한 리터럴만 잡는다 → 한 줄짜리 메뉴 라벨·버튼·푸터·제목이 전부 안 보인다. 미래핑 390개가 정확히 그 형태다
2. `WATCHED` 가 `src/render`, `src/core` 대부분, `src/mods` 대부분, `src/link/Protocol.lua`·`Handshake.lua` 를 누락
3. `EXEMPT_FILES` 가 `src/ui/OakSpeech.lua` 를 통째로 면제 — 그런데 `:364`("HIS NAME?"), `:419`("YES"/"NO") 는 텍스트 id 폴백이 아니라 엔진 저작 UI 다

- [ ] **Step 1: 현재 게이트를 읽는다**

```sh
sed -n '1,140p' tests/engine/gate_strings_coverage.lua
```

`WATCHED_DIRS`, `WATCHED_FILES`, `ALLOWED`, `EXEMPT_FILES`, `offenders()` 의 정확한 형태를 파악한다.

- [ ] **Step 2: `\n` 요구를 제거하고 얼마나 늘어나는지 측정**

`offenders()` 에서 리터럴이 `\n`/`\f`/`\v` 를 포함해야 한다는 조건을 뺀 뒤:

```sh
luajit tests/engine/gate_strings_coverage.lua 2>&1 | tail -30
```

**이 숫자를 기록한다.** 스펙은 390개를 예상한다. 실제 숫자가 계획의 나머지를 정한다.

- [ ] **Step 3: 게이트를 램프 방식으로 만든다**

390개를 한 번에 고칠 수 없으므로, **현재 위반 수를 상한으로 고정**하고 그 이상 늘어나면 실패하게 한다. 파일 단위 baseline 을 파일에 적어두고, 새 위반이 생기면 게이트가 잡는다.

```lua
-- Ratchet: today's offender count per file.  A new unwrapped literal fails
-- the gate; a fixed one must be removed from this table, so the number can
-- only go down.  See docs/superpowers/plans/2026-08-01-korean-localization-phase-0-1.md
local BASELINE = {
  -- ["src/ui/ListMenu.lua"] = 12,   <- Step 2 의 실측값으로 채운다
}
```

- [ ] **Step 4: WATCHED 확장 + OakSpeech 면제 축소**

`WATCHED_DIRS` 에 `src/render`, `src/core`, `src/mods`, `src/link` 를 추가하고, `EXEMPT_FILES` 의 `src/ui/OakSpeech.lua` 통째 면제를 제거한 뒤 `_OakSpeech*` 접두 리터럴만 `ALLOWED` 규칙으로 예외 처리한다. baseline 을 다시 측정해 갱신한다.

- [ ] **Step 5: 게이트 통과 확인**

```sh
luajit tests/engine/gate_strings_coverage.lua
./scripts/test.sh 2>&1 | tail -5
```

- [ ] **Step 6: 커밋**

```sh
git add tests/engine/gate_strings_coverage.lua
git commit -m "test: widen the strings coverage gate and ratchet it

The gate only flagged literals containing \\n/\\f/\\v, so every single-line
menu label, button caption and footer was invisible to it -- which is what
the unwrapped literals are made of. Widens WATCHED, drops the wholesale
OakSpeech exemption, and pins today's per-file counts so the number can
only go down."
git push -u origin HEAD
```

> **업스트림 PR #3.**

---

# Phase 1 — 화면에 한글이 나온다

**이 단계가 끝나면:** 한글 UI 로 실제 플레이가 가능하고, 8x16 가독성을 실기에서 판단할 수 있다.

---

### Task 8: 한글 글리프 시트 생성기

**Files:**
- Create: `ko-tools/build_sheet.py`
- Create: `ko-tools/README.md`

시트는 **엔진 저장소가 아니라** `ko-tools/` 에서 만든다. 산출물은 번역 모드로 들어간다.

**Interfaces:**
- Produces: `assets/font/ko.png` (1024×1408, 8px 폭 × 16px 높이 셀, 128/행, 완성형 11,172자) 와 `lang/charmap.lua` (`{ seq = "가", code = 0x100 }` 형태)

- [ ] **Step 1: 폰트를 받고 메트릭을 확인**

```sh
mkdir -p ko-tools/fonts && cd ko-tools/fonts
curl -sLO https://github.com/quiple/galmuri/releases/download/v2.40.4/Galmuri-v2.40.4.zip
unzip -oq Galmuri-v2.40.4.zip
python3 - <<'PY'
from fontTools.ttLib import TTFont
f = TTFont("Galmuri11-Condensed.ttf")
upem = f["head"].unitsPerEm
asc, desc = f["hhea"].ascent, f["hhea"].descent
adv = f["hmtx"][f.getBestCmap()[0xAC00]][0]
print("upem", upem, "asc", asc, "desc", desc, "hangul adv", adv)
print("ppem for 8px advance:", round(8 * upem / adv))
print("cell height at that ppem:", round((asc - desc) * round(8*upem/adv) / upem))
PY
```

기대: `upem 1200 asc … hangul adv 800`, `ppem for 8px advance: 12`, `cell height … 14`. 셀은 16px 로 패딩한다.

- [ ] **Step 2: 시트 생성기를 쓴다**

`ko-tools/build_sheet.py`:

```python
#!/usr/bin/env python3
"""완성형 한글 11,172자를 8px 폭 x 16px 높이 셀 시트로 굽는다.

엔진은 한 음절 = 글리프 1개로 그린다 (charmap 이 바이트열 -> 글리프 코드
1개 고정). 조합형은 런타임에 불가능하므로 여기서 미리 완성형으로 굽는다.
"""
import json, sys
from pathlib import Path
from PIL import Image, ImageDraw, ImageFont
from fontTools.ttLib import TTFont

HERE = Path(__file__).resolve().parent
FONT = HERE / "fonts" / "Galmuri11-Condensed.ttf"
BASE = 0x100          # 문서상 자유 영역 (바닐라 페이지는 0x60/0x80)
PER_ROW = 128
CELL_W, CELL_H = 8, 16
FIRST, LAST = 0xAC00, 0xD7A3   # 완성형 11,172자


def ppem_for_advance(path, want_px):
    f = TTFont(path)
    upem = f["head"].unitsPerEm
    adv = f["hmtx"][f.getBestCmap()[0xAC00]][0]
    asc = f["hhea"].ascent
    return round(want_px * upem / adv), upem, asc


def main():
    ppem, upem, asc = ppem_for_advance(FONT, CELL_W)
    asc_px = round(asc * ppem / upem)
    font = ImageFont.truetype(str(FONT), ppem)

    syllables = [chr(c) for c in range(FIRST, LAST + 1)]
    rows = (len(syllables) + PER_ROW - 1) // PER_ROW
    sheet = Image.new("RGBA", (PER_ROW * CELL_W, rows * CELL_H), (0, 0, 0, 0))

    for i, ch in enumerate(syllables):
        cell = Image.new("L", (CELL_W * 3, CELL_H), 255)
        # anchor "ls" = left/baseline; sit the baseline where the 16px cell
        # wants it so Hangul and the vanilla 8px Latin share a bottom edge
        ImageDraw.Draw(cell).text((0, CELL_H - (CELL_H - asc_px)), ch,
                                  font=font, fill=0, anchor="ls")
        cell = cell.point(lambda v: 0 if v < 128 else 255)   # 1-bit, no AA
        x, y = (i % PER_ROW) * CELL_W, (i // PER_ROW) * CELL_H
        for py in range(CELL_H):
            for px in range(CELL_W):
                if cell.getpixel((px, py)) == 0:
                    sheet.putpixel((x + px, y + py), (0, 0, 0, 255))

    out_png = HERE / "out" / "ko.png"
    out_png.parent.mkdir(exist_ok=True)
    sheet.save(out_png, optimize=True)

    lines = ["-- generated by ko-tools/build_sheet.py -- do not hand-edit",
             "return {"]
    for i, ch in enumerate(syllables):
        lines.append('  { seq = "%s", code = %d },' % (ch, BASE + i))
    lines.append("}")
    (HERE / "out" / "charmap.lua").write_text("\n".join(lines), encoding="utf-8")

    print(f"ko.png {sheet.width}x{sheet.height}  {out_png.stat().st_size} bytes")
    print(f"charmap.lua {len(syllables)} entries, base 0x{BASE:X}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 3: 돌려서 산출물 크기를 확인**

```sh
cd ko-tools && python3 build_sheet.py
```

기대: `ko.png 1024x1408`, 파일 크기 200KB 미만. 아니면 `PER_ROW`/패딩을 재검토한다.

- [ ] **Step 4: 눈으로 검수**

```sh
python3 - <<'PY'
from PIL import Image
im = Image.open("out/ko.png")
# 받침이 많은 대표 음절들을 잘라 확대
FIRST = 0xAC00
for ch in "김철수몸났뷁을":
    i = ord(ch) - FIRST
    x, y = (i % 128) * 8, (i // 128) * 16
    im.crop((x, y, x+8, y+16)).resize((80,160), Image.NEAREST).save(f"out/chk_{ch}.png")
print("wrote out/chk_*.png")
PY
```

받침이 뭉개지지 않았는지 확인한다.

- [ ] **Step 5: 커밋**

```sh
cd /Users/hwanmooy/Dropbox/dev/pokemon/ko-tools
git init -q && git add -A && git commit -q -m "feat: Korean glyph sheet generator (8x16, Galmuri11 Condensed)"
```

> `ko-tools/out/` 와 `ko-tools/fonts/` 는 `.gitignore` 에 넣는다 (산출물·서드파티 폰트).

---

### Task 9: `Font.lua` 페이지별 셀 높이

**Files:**
- Modify: `src/render/Font.lua:14,38-53,187-192`
- Modify: `src/mods/Schemas.lua` (`R.font` 에 `cellH`)
- Test: `tests/parity_font_cell_height.lua` (신규)

**Interfaces:**
- Produces: `Font.cellHeightOf(code) -> number`, `Font.maxCellHeight() -> number`. Task 10 이 사용한다.

- [ ] **Step 1: 실패하는 테스트를 쓴다**

`tests/parity_font_cell_height.lua`:

```lua
-- A font page may declare a taller cell (Korean needs 8x16). The glyph grows
-- UPWARD from the same baseline row, so nothing that owns an 8px row has to
-- move -- and a page that does not declare cellH must render byte-identically.
package.path = "./?.lua;./?/init.lua;" .. package.path
if not _G.love then _G.love = require("tests.love_stub") end

local S = require("tests.harness").suite("font cell height")
local check, eq = S.check, S.eq

local Font = require("src.render.Font")

-- record what the love stub is asked to draw
local drawn = {}
local realDraw = love.graphics.draw
love.graphics.draw = function(img, quad, x, y) drawn[#drawn+1] = { x = x, y = y } end

Font.load({ font = { pages = {
  main = { image = "assets/generated/font/main.png", base = 0x60, glyphsPerRow = 16 },
  ko   = { image = "assets/generated/font/ko.png",   base = 0x100,
           glyphsPerRow = 128, cellH = 16 },
} } })

eq(Font.cellHeightOf(0x60), 8, "a page without cellH is 8 tall")
eq(Font.cellHeightOf(0x100), 16, "a page with cellH = 16 reports 16")
eq(Font.maxCellHeight(), 16, "maxCellHeight is the tallest loaded page")

drawn = {}
Font.drawCode(0x60, 40, 112)
eq(drawn[1] and drawn[1].y, 112, "an 8px glyph draws at the y it was given")

drawn = {}
Font.drawCode(0x100, 40, 112)
eq(drawn[1] and drawn[1].y, 104,
  "a 16px glyph is lifted 8px so its baseline row is unchanged")

love.graphics.draw = realDraw
S.finish()
```

- [ ] **Step 2: 테스트를 돌려 실패를 확인**

```sh
luajit tests/parity_font_cell_height.lua
```

기대: `Font.cellHeightOf` 없음으로 에러.

- [ ] **Step 3: 구현**

`src/render/Font.lua`:

```lua
local GLYPH = 8   -- cell WIDTH + advance; still the baseline row pitch
```

`Font.load` 의 페이지 루프:

```lua
      local iw, ih = img:getDimensions()
      -- per-page cell height; a page that declares cellH = 16 draws glyphs
      -- that grow UPWARD from the same 8px baseline row
      local cellH = page.cellH or GLYPH
      local perRow = page.glyphsPerRow or math.floor(iw / GLYPH)
      local quads = {}
      for i = 0, perRow * math.floor(ih / cellH) - 1 do
        quads[i] = love.graphics.newQuad((i % perRow) * GLYPH,
          math.floor(i / perRow) * cellH, GLYPH, cellH, iw, ih)
      end
      local entry = { id = id, image = img, quads = quads,
                      base = page.base, cellH = cellH,
                      advance = page.advance or GLYPH }
```

`Font.drawCode`:

```lua
function Font.drawCode(code, x, y)
  local page = pageFor(code)
  if not page then return end
  local quad = page.quads[code - page.base]
  -- absorb the extra height upward so the glyph keeps its baseline row
  if quad then love.graphics.draw(page.image, quad, x, y - (page.cellH - GLYPH)) end
end

-- the cell height a code draws with; 8 unless its page says otherwise.
-- Callers that own vertical layout must consult this, not assume 8.
function Font.cellHeightOf(code)
  local page = pageFor(code)
  return page and page.cellH or GLYPH
end

-- the tallest cell among loaded pages.  Vanilla is 8, so every layout
-- decision keyed on this is a no-op until a taller page is registered.
function Font.maxCellHeight()
  local tallest = GLYPH
  for _, page in ipairs(state.order) do
    if page.cellH > tallest then tallest = page.cellH end
  end
  return tallest
end
```

`src/mods/Schemas.lua` 의 `R.font` 페이지 스키마에 `cellH` 를 정수 필드로 추가한다 (`base`, `glyphsPerRow` 옆).

- [ ] **Step 4: 테스트 통과 + 영문 회귀 확인**

```sh
luajit tests/parity_font_cell_height.lua
./scripts/test.sh 2>&1 | tail -5
```

기대: 둘 다 통과. **바닐라 페이지는 `cellH == 8` 이라 오프셋이 0 이므로 영문은 바이트 동일해야 한다.**

- [ ] **Step 5: 커밋**

```sh
git add src/render/Font.lua src/mods/Schemas.lua tests/parity_font_cell_height.lua
git commit -m "feat(font): per-page glyph cell height

GLYPH = 8 was the quad's width AND height, so a script needing a taller
cell -- Korean at 8x16, and Japanese kanji for the same reason -- could not
be described to the engine at all. A page may now declare cellH; the glyph
grows upward from the same baseline row, so the ~350 draw sites are
untouched and a page without cellH renders byte-identically."
git push -u origin HEAD
```

> **업스트림 PR #4.** 이슈 #245(일본어)가 같은 벽에 막혀 있으므로 **PR 을 열기 전에 Discord 에 물어본다** — 이미 진행 중일 수 있다.

---

### Task 10: 16px 글리프에 맞춘 세로 레이아웃

**Files:**
- Modify: `src/render/TextBox.lua:172,306,319`
- Modify: `src/battle/BattleState.lua:868,4985,5017,5045`
- Modify: `src/battle/WideBattle.lua:152`
- Modify: `src/ui/SummaryMenu.lua:156-157,204-206`, `src/ui/DexEntryMenu.lua:101-105`
- Test: `tests/parity_text_scroll_bounds.lua` (기존), `tests/parity_tall_glyph_layout.lua` (신규)

**핵심 제약:** 스크롤 상수를 무조건 8→16 으로 바꾸면 **영문 동작이 바뀐다.** `Font.maxCellHeight()` 로 분기해야 한다.

- [ ] **Step 1: 현재 영문 스크롤 동작을 먼저 못박는 테스트를 쓴다**

`tests/parity_tall_glyph_layout.lua`:

```lua
-- The 16px-glyph layout must be invisible to English. These cases pin the
-- vanilla scroll geometry first, so a change that "fixes" Korean by moving
-- English fails loudly.
package.path = "./?.lua;./?/init.lua;" .. package.path
if not _G.love then _G.love = require("tests.love_stub") end

local S = require("tests.harness").suite("tall glyph layout")
local eq = S.eq

local Font = require("src.render.Font")
local TextBox = require("src.render.TextBox")

-- vanilla: only 8px pages loaded
Font.load({ font = { pages = {
  main = { image = "assets/generated/font/main.png", base = 0x60, glyphsPerRow = 16 },
} } })
eq(Font.maxCellHeight(), 8, "vanilla font is 8px")
eq(TextBox.lineStep(), 8, "vanilla scroll step is unchanged")

-- with a 16px page registered
Font.load({ font = { pages = {
  main = { image = "assets/generated/font/main.png", base = 0x60, glyphsPerRow = 16 },
  ko   = { image = "assets/generated/font/ko.png", base = 0x100,
           glyphsPerRow = 128, cellH = 16 },
} } })
eq(Font.maxCellHeight(), 16, "a 16px page raises maxCellHeight")
eq(TextBox.lineStep(), 16, "scroll step follows the tallest page")

S.finish()
```

- [ ] **Step 2: 테스트를 돌려 실패를 확인**

```sh
luajit tests/parity_tall_glyph_layout.lua
```

기대: `TextBox.lineStep` 없음으로 에러.

- [ ] **Step 3: `TextBox.lineStep()` 을 추가하고 상수를 대체**

`src/render/TextBox.lua`:

```lua
-- Scroll travel for one retired line. Vanilla glyphs are 8px so this is 8,
-- exactly as it always was; a taller font page (Korean at 16) needs the full
-- row pitch or the incoming line lands on top of the outgoing one.
function TextBox.lineStep()
  return Font.maxCellHeight()
end
```

`:172` 의 `self.scrollPx = 8` 을 `self.scrollPx = TextBox.lineStep()` 으로,
`:306` 의 `self.scrollPx = self.scrollPx - 2` 를 `self.scrollPx - TextBox.lineStep() / 4` 로 바꾼다 (4프레임 슬라이드 유지).

`:319` 는 8px 폰트에서 동작이 바뀌면 안 되므로 조건부로 쓴다:

```lua
    -- Both lines carry the offset once glyphs are taller than the 8px row:
    -- a retained line at line1Y+off would otherwise overlap the incoming
    -- line's band. At 8px this is the historical behaviour, unchanged.
    local tall = Font.maxCellHeight() > 8
    local y = (ys[i] or self.line2Y) + ((tall or i == 1) and off or 0)
```

- [ ] **Step 4: 테스트 통과 + 영문 스크롤 회귀 확인**

```sh
luajit tests/parity_tall_glyph_layout.lua
luajit tests/parity_text_scroll_bounds.lua
./scripts/test.sh 2>&1 | tail -5
```

기대: 전부 통과. **`parity_text_scroll_bounds` 가 깨지면 영문을 건드린 것이다.**

- [ ] **Step 5: 배틀 스크롤도 같은 방식으로**

`src/battle/BattleState.lua:868,4985` 와 `src/battle/WideBattle.lua:152` 의 하드코딩 `8`/`2` 를 `TextBox.lineStep()` 기준으로 바꾼다.

- [ ] **Step 6: 8px 세로 피치 화면 4곳을 16px 로 재배치**

| 화면 | 위치 | 변경 |
|---|---|---|
| 배틀 기술 목록 | `BattleState.lua:5017` | `96 + i * 8` → `96 + i * Font.maxCellHeight()` |
| Mimic 목록 | `BattleState.lua:5045` | `(7 + i) * 8` → 같은 방식 |
| 요약 화면 | `SummaryMenu.lua:156-157,204-206` | `y` / `y + 8` 쌍을 `y` / `y + Font.maxCellHeight()` 로 |
| 도감 항목 | `DexEntryMenu.lua:101-105` | `y=44` / `y=54` 를 폰트 높이 기준으로 |

각 변경 후 `./scripts/test.sh` 로 영문 회귀를 확인한다.

- [ ] **Step 7: 커밋**

```sh
git add -A src/ tests/parity_tall_glyph_layout.lua
git commit -m "feat(layout): follow the font's cell height for vertical pitch

Screens that stack text rows 8px apart collide once a glyph is 16px tall.
Everything keys off Font.maxCellHeight(), which is 8 for vanilla, so English
output is unchanged -- pinned by parity_tall_glyph_layout and the existing
scroll-bounds parity suite."
git push -u origin HEAD
```

> **업스트림 PR #5** (PR #4 에 의존).

---

### Task 11: 번역 모드 스캐폴드 + 한글이 화면에 뜬다

**이 태스크가 이 계획의 목표점이다.** 끝나면 게임에 한글이 실제로 렌더된다.

**Files:**
- Create: `gen1recomp-ko/` (스캐폴드 산출물)

- [ ] **Step 1: 스캐폴드**

```sh
cd /Users/hwanmooy/Dropbox/dev/pokemon/gen1recomp
.venv/bin/python3 tools/modkit.py translation korean --language 한국어 \
  --dest /Users/hwanmooy/Dropbox/dev/pokemon
```

기대 출력에 `lang/dialogue.lua 2582 entries`, `lang/strings.lua 576 entries` 가 보여야 한다.

- [ ] **Step 2: 시트와 charmap 을 모드에 넣는다**

```sh
cd /Users/hwanmooy/Dropbox/dev/pokemon
mkdir -p korean/assets/font
cp ko-tools/out/ko.png korean/assets/font/ko.png
cp ko-tools/out/charmap.lua korean/lang/charmap.lua
```

`korean/lang/font.lua` 를 아래로 채운다:

```lua
return {
  ko = {
    image = "assets/font/ko.png",
    base = 0x100,
    glyphsPerRow = 128,
    cellH = 16,
  },
}
```

- [ ] **Step 3: 한 줄만 번역해서 렌더를 확인한다**

`korean/lang/strings.lua` 에서 짧고 확실한 키 하나만 번역한다. 예:

```lua
  ["POKéDEX"] = "도감",
```

- [ ] **Step 4: 게임을 띄워 눈으로 확인**

```sh
cd gen1recomp && POKEPORT_AUTOPILOT=1 love . 2>&1 | tail -20
```

스크린샷은 LÖVE 저장 디렉토리에 떨어진다. **"도감" 이 깨지지 않고 보이면 이 계획의 핵심 가설이 검증된 것이다.** 안 보이면:
- 로그에 `font: no glyph for` 가 있으면 charmap 문제
- 아무 로그도 없이 빈칸이면 **Task 3 의 폰트 경로 문제** — `mod.assets:path` 가 적용됐는지 확인

- [ ] **Step 5: 커밋**

```sh
cd /Users/hwanmooy/Dropbox/dev/pokemon/korean
git init -q && git add -A
git commit -q -m "feat: Korean translation mod scaffold with 8x16 glyph sheet"
```

---

### Task 12: 고유명사 사전

**Files:**
- Modify: `ko-tools/fetch_terms.py` (별칭 맵 추가)
- Create: `ko-tools/terms_manual.json`
- Create: `ko-tools/emit_catalogs.py`
- Modify: `korean/lang/species_names.lua`, `move_names.lua`, `item_names.lua`, `trainer_names.lua`

**현황 (실측):** PokéAPI 로 종족 151/151, 기술 165/165, 타입 18/18, 아이템 125/152 확보. `ko-tools/terms.json` 에 이미 있다.

- [ ] **Step 1: 자동 매핑 오류를 검수한다**

**PokéAPI 는 세대 구분 없이 최신 명칭을 준다.** 확인된 오매핑:

| 키 | 잘못된 값 | 실제 |
|---|---|---|
| `COIN` | 동전케이스 | COIN CASE 의 이름. COIN 은 별개 아이템 |
| `EXP_ALL` | 학습장치 | EXP.SHARE 의 이름. 1세대 EXP.ALL 은 파티 전체 분배 |
| `ITEMFINDER` | 다우징머신 | 4세대 개명. 2세대 정식 한국어명 확인 필요 |

`terms.json` 을 전수 훑어 이런 것을 걸러낸다. 기준: **1세대/2세대 한국어 정식 명칭**.

- [ ] **Step 2: 수동 사전을 만든다**

`ko-tools/terms_manual.json` — PokéAPI 에 없는 것들:
- 뱃지 8개 (`BOULDERBADGE` … `EARTHBADGE`)
- 엘리베이터 층 표시 14개 (`FLOOR_1F` … `FLOOR_B4F`) — "1F"/"B1F" 그대로 두면 된다
- `BIKE_VOUCHER`, `POKEDEX`, `SURFBOARD`
- 트레이너 클래스 47개
- 지명 ~226개

- [ ] **Step 3: 카탈로그 생성기를 쓴다**

`ko-tools/emit_catalogs.py` 가 `terms.json` + `terms_manual.json` + 별칭맵을 합쳐 `korean/lang/*_names.lua` 를 쓴다. 형식은 스캐폴더가 만든 것과 동일하게 맞춘다 (`["ENGLISH_KEY"] = "한국어",`).

- [ ] **Step 4: 게임에서 확인**

```sh
cd gen1recomp && POKEPORT_AUTOPILOT=1 love . 2>&1 | tail -20
```

포켓몬 이름이 한글로 나오는지 확인한다.

- [ ] **Step 5: 커밋**

```sh
cd ko-tools && git add -A && git commit -q -m "feat: official Korean terminology pipeline"
cd ../korean && git add -A && git commit -q -m "feat: Korean proper nouns"
```

---

### Task 13: 엔진 문자열 576개

**Files:**
- Modify: `korean/lang/strings.lua`

- [ ] **Step 1: 미번역 목록을 뽑는다**

```sh
cd gen1recomp
.venv/bin/python3 tools/modkit.py translation korean --language 한국어 \
  --dest /Users/hwanmooy/Dropbox/dev/pokemon --refresh
```

`--refresh` 가 남은 키를 보고한다. **Task 4·5 가 끝난 뒤여야 안전하다.**

- [ ] **Step 2: 배치로 번역한다**

Task 12 의 용어 사전을 강제 참조한다. 규칙:
- **포맷 지정자 개수를 바꾸지 않는다.** `Strings.lua:103` 의 arity 가드가 개수가 다르면 번역을 버리고 영어를 그린다
- **인자 순서를 바꿀 수 없다.** LuaJIT `string.format` 에 위치 지정자가 없다. 어순이 안 맞는 82곳은 Phase 2 의 josa 패치에서 해결하므로, 지금은 **자연스럽지 않아도 순서를 지킨다**
- `\n`(줄바꿈) / `\v`(스크롤) / `\f`(페이지) 제어문자를 보존한다
- 한 줄 18글자를 넘지 않게 끊는다

- [ ] **Step 3: 줄 길이를 기계적으로 검증**

```sh
cd ko-tools
python3 - <<'PY'
import re, sys
bad = 0
for line in open("../korean/lang/strings.lua", encoding="utf-8"):
    m = re.match(r'\s*\["(.*)"\]\s*=\s*"(.*)",\s*$', line)
    if not m: continue
    for seg in re.split(r'\\n|\\v|\\f', m.group(2)):
        seg = seg.replace('\\"', '"')
        if len(seg) > 18:
            print(f"{len(seg)}자: {seg}")
            bad += 1
print("over-long segments:", bad)
sys.exit(1 if bad else 0)
PY
```

- [ ] **Step 4: 게임에서 확인 후 커밋**

```sh
cd ../gen1recomp && ./scripts/test.sh 2>&1 | tail -3
cd ../korean && git add -A && git commit -q -m "feat: Korean engine strings"
```

---

### Task 14: 실기 가독성 확인 — Phase 2 의 입력

**이것이 Phase 1 의 종료 조건이자, Phase 2 계획의 전제다.**

- [ ] **Step 1: 데스크톱에서 확인**

```sh
cd gen1recomp && scripts/run.sh
```

대화상자, 배틀 메시지, 메뉴, 요약 화면을 돌아다니며 받침 많은 글자(몸, 났, 을, 뷁)를 본다.

- [ ] **Step 2: 핸드헬드 빌드로 확인**

```sh
./build-rg34xxsp.sh
```

RG34XXSP 실물 패널에서 본다. **닌텐도가 한국어 금·은을 GBC 전용으로 낸 이유가 이 지점이다.**

- [ ] **Step 3: 판정을 기록한다**

`docs/superpowers/specs/2026-08-01-korean-localization-design.md` 의 "남은 위험" 1번을 결과로 갱신하고 커밋한다.

- **읽을 만하다** → Phase 2 (josa + 위치 지정 인자 + 미래핑 390개) 로 진행
- **안 읽힌다** → 글리프 기하를 재검토한다. `cellH` 는 이미 있으므로 시트만 바꾸면 되고, 폭을 늘리려면 `Font.drawCode` 의 8px stride 와 줄바꿈 예산까지 다시 봐야 한다

---

## Self-Review

**스펙 커버리지**

| 스펙 항목 | 태스크 |
|---|---|
| P1 `cellH` | Task 9 |
| P2 modkit 버그 5종 + 테스트 | Task 1–5 |
| P3 미래핑 문자열 | Task 7 (게이트만). **본체는 Phase 2** — 스펙의 단계표대로다 |
| P4 세로 피치 | Task 10 |
| P5 josa | **Phase 2** — 스펙대로 |
| P6 이름 바이트 절단 | Task 6 |
| 폰트 시트 | Task 8 |
| 고유명사 | Task 12 |
| 엔진 문자열 | Task 13 |
| 이름 입력 IME | **Phase 4** — 스펙대로 |
| 실기 확인 | Task 14 |

**남은 구멍:** `korean/manifest.json` 에 다른 언어팩과의 `conflicts` 를 넣는 단계가 Task 11 에 명시돼 있지 않다. Step 2 에서 함께 처리한다:

```json
  "conflicts": [],
  "category": "LANGUAGE"
```

다른 언어팩 id 가 알려지면 `conflicts` 에 추가한다. 오버라이드는 last-writer-wins 라 충돌 감지가 없으므로, 한국어+다른 언어를 같이 켜면 조용히 섞인다.

**타입 일관성:** `Font.glyphCount`/`Font.cut` (Task 6), `Font.cellHeightOf`/`Font.maxCellHeight` (Task 9), `TextBox.lineStep` (Task 10) — 이름이 태스크 간에 일치한다.
