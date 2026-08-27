# Learning journal — building a local Nginx + Spring Boot + Postgres stack

This is a record of an assignment from a backend development course: set up
Nginx locally, serve a static site through it, reverse-proxy it to a dynamic
app, then wire that app to a real database — and be able to explain every
step, not just get it working. The instructor's condition was explicit: no
AI writing the code — guidance and step-by-step review only, mistakes and
all. This document is the "all" part: the wrong turns are kept in, because
they're what actually built the mental model.

It's organized around the seven parts of the original assignment. Each part
ends with a **Confusions resolved** section — the specific misconceptions
that came up, and the correction, written the way they actually happened.

---

## Part 1 — Install and run Nginx

Installed the Windows Nginx build, extracted it (into an accidentally
double-nested folder — `nginx-1.30.4/nginx-1.30.4/`, worth checking before
assuming a command like `.\nginx.exe` will find the binary), and ran
`nginx.exe` from inside that folder. Confirmed the default welcome page at
`http://localhost`.

**Confusions resolved**

| ❌ Misconception | ✅ Reality |
|---|---|
| `nginx.exe -s reload` fails, so Nginx must be broken or missing | PowerShell (unlike `cmd.exe`) won't run an executable from the current directory by name alone. Fix: `.\nginx.exe -s reload`. |
| Two `nginx.exe` processes in Task Manager means something's wrong | Correct, expected state: one **master process** (reads config, handles `-s reload`) + one **worker process** (actually serves requests). |
| Nginx is redundant since Spring Boot's embedded Tomcat is already a full HTTP server | Tomcat *can* serve static files, but every request — even an image — passes through the full servlet lifecycle, wasted overhead for "read file, send bytes." Nginx does that one job cheaper, and in production also acts as the single public entry point, TLS terminator, and load balancer in front of app servers. |

## Part 2 — Serve a static site through Nginx

Wrote `index.html`, `style.css`, `script.js` by hand and pointed Nginx's
`location / { root ...; index index.html; }` at that folder.

**Confusions resolved**

| ❌ Misconception | ✅ Reality |
|---|---|
| Files saved from a plain text editor are named `index.html` | Windows hides known extensions by default — they were actually `index.html.txt`. Fixed via File Explorer → View → show file name extensions. |
| `<head/>` is a valid way to close a tag | HTML doesn't self-close container tags that way — needs `</head>`. |
| Visible page text belongs inside `<head>` | `<head>` is for metadata only (title, `<link>`); visible content goes in `<body>`. |
| `ref="..."` links a `<script>`/`<link>` to a file | Not a real attribute — it's `src` for `<script>`, `href` for `<link>`. |
| `<script src="..." />` is valid (self-closing) | `<script>` always needs an explicit closing tag, since it can hold inline code between the tags. |
| The last CSS declaration in a block doesn't need a `;` | Every declaration needs one, including the last — easy to miss since the block still "looks" closed without it. |
| `16sp` is a valid CSS font-size unit | That's Android's unit. CSS uses `px`, `em`, `rem`. |
| `func updateP() { }` declares a JS function | `func` is Swift/Go. JS uses `function`. |
| `p: { onchange = Text("...") }` updates an element's text | Not real JS. Actual pattern: `document.querySelector("p").textContent = "..."`. |
| `.onClick = fn` wires up a click handler | Case-sensitive — it's `.onclick`, all lowercase. |
| "I fixed the file" but the browser shows no change | The save often hadn't actually landed on disk — always verify the file changed before assuming a fix took effect. |

## Part 3 — A Spring Boot "Hello World"

Generated a project via Spring Initializr (Maven, Java, `Spring Web` only),
wrote one `@RestController` with a `GET /hello` endpoint returning a plain
string, ran it, and hit `http://localhost:8080/hello` directly.

**Confusions resolved**

| ❌ Misconception | ✅ Reality |
|---|---|
| `@RestController` means "conforming to a REST protocol" | REST isn't a protocol — it's an architectural style built on HTTP, the actual protocol. `@RestController` = `@Controller` (handles requests) + `@ResponseBody` (return value goes straight into the HTTP response body, instead of being resolved as an HTML view template name). |
| `@Controller` is just a worse version of `@RestController` | Different purpose: `@Controller` is for server-rendered pages via a template engine — a legitimate, different architecture. |
| Initializr's Java dropdown should show your exact installed version (23) | It may only list a few options. Rule: pick the highest listed version ≤ your installed one (21 here) — a newer JDK runs older-targeted code fine. |
| `mvn spring-boot:run` failing means Maven/the project is broken | Usually just means Maven isn't installed globally. Spring Initializr projects ship a **Maven Wrapper** — `.\mvnw.cmd spring-boot:run` needs no global install. |
| The endpoint doesn't work — no text shows in the browser | Was actually a broken/sandboxed test browser, not the app. A real browser (Edge) showed it working immediately. Rule out the test environment before assuming the app is broken. |

## Part 4 — Reverse proxy Nginx → Spring Boot

Added a second `location` block to the same Nginx `server {}`, using
`proxy_pass` to forward matching requests to the Spring Boot app on port
8080.

**Confusions resolved** — this was the deepest rabbit hole. The trailing-slash
mechanism especially isn't obvious from the directive names alone, so it's
worth a dedicated table rather than prose.

**Why a second `location` block at all?** Not because `proxy_pass` can't
target port 8080 from `location /` (it can, from any block) — but because
`location /` already has a job (`root`, serving static files), and one block
can't serve files *and* proxy for the same path without conflicting.

**The trailing-slash mechanism** — worked out by testing every combination
directly against `GET /api/hello` (controller mapped to `@GetMapping("/hello")`):

| `location` | `proxy_pass` | What Spring Boot receives | Result |
|---|---|---|---|
| `/api/` | `http://localhost:8080/` | `/hello` (prefix stripped, remainder appended onto proxy_pass's `/`) | ✅ 200 — correct, and works for *any* sub-path |
| `/api/` | `http://localhost:8080` *(no path at all)* | `/api/hello` *(nothing stripped — no anchor to append onto, so the full original path is forwarded unchanged)* | ❌ 404 (proxy hop worked, wrong path reached Spring Boot) |
| `/api` *(no slash)* | `http://localhost:8080/` | `//hello` *(remainder keeps its own leading slash: `/hello`, appended onto proxy_pass's `/`)* | ❌ Double slash — works by accident for the bare `/api` request, breaks silently on any sub-path |

**Takeaway:** keep the trailing slash consistent on **both** `location` and
`proxy_pass` — it's the only combination that behaves predictably for every
sub-path, not just the one you happened to test first. The `/` in
`proxy_pass http://localhost:8080/;` is doing real work (it's the anchor the
stripped prefix gets appended onto) — it's not decorative.

| ❌ Misconception | ✅ Reality |
|---|---|
| Editing `nginx.conf` takes effect immediately | A running worker holds its config already parsed in memory. Nothing changes until `nginx -s reload` tells the master to re-read the file and start new workers with it. |
| `reload` and `stop` are basically the same | `reload` is **graceful**: old workers finish in-flight requests before exiting, new workers (new config) start accepting immediately — zero dropped connections. Same principle as zero-downtime rolling deploys. |
| Adding Nginx changed what Tomcat *is* | Tomcat was always both a web server and an application server. What changed is *routing* — static requests now never reach it. "Nginx = web server, Tomcat = app server" describes this architecture's division of labor, not a hard limit on either. |
| `localhost` needs a working network connection | `127.0.0.1` (loopback) is handled entirely inside the OS's own stack — never touches a physical adapter, so it works with WiFi off. Source and destination being the same machine is literally what "loopback" means. |
| Client/server are fixed identities | They're roles. The same Spring Boot process is a *server* to the browser and, once it calls Postgres, simultaneously a *client* to the database. Long backend chains are this same relationship repeated at each hop. |
| A client needs a known, fixed port like a server does | Only servers (Postgres, Nginx, Tomcat) *listen* on a well-known port so others can find them. A client (TablePlus, or Spring Boot as a DB client) gets a disposable, OS-assigned port per connection — nobody needs to know it, since the client initiates contact. |

## Part 5 — Local database via DBngin + TablePlus

Installed DBngin, spun up a local Postgres instance (host `127.0.0.1`, port
`5432`, user `postgres`, no password), installed TablePlus, connected to it,
and created a `products` table (`id`, `name`, `price`) with two rows inserted
by hand.

**Confusions resolved**

| ❌ Misconception | ✅ Reality |
|---|---|
| DBngin stalled or failed when "start" triggered a download | Expected — DBngin doesn't bundle database engines, it downloads the real Postgres binaries on first use of a given instance. |
| The DB is running but nothing visible appears — must be broken | Postgres has no UI of its own; it just speaks a wire protocol over its port (same as Nginx having no window). TablePlus is the **client** that connects and renders a visual grid. |
| DBngin *is* the database | DBngin is a manager/GUI for downloading and starting real engines. Postgres and MySQL are the actual databases — peer implementations DBngin can launch either of. |
| `INSERT` fails with `column "name" ... does not exist`, right after adding it via the GUI | The GUI table-builder had **staged** the columns (shown in a different color) but never committed them to Postgres. Fixed by dropping the table and recreating it in one atomic `CREATE TABLE` via SQL instead. |
| No `public` schema visible in TablePlus's sidebar means something's missing | Postgres creates `public` automatically with every database — TablePlus just collapses that layer visually when there's only the one default schema, showing "Tables" directly. |

## Part 6 — Connect Spring Boot to the database

Added `spring-boot-starter-data-jpa` and the `org.postgresql:postgresql`
driver to `pom.xml`, configured `application.properties`, wrote a `Product`
`@Entity` matching the existing table, an empty `ProductRepository`
interface extending `JpaRepository`, and a `GET /products` endpoint that
calls `.findAll()`.

**Confusions resolved**

| ❌ Misconception | ✅ Reality |
|---|---|
| `<artifactId>org.postgresql:postgresql</artifactId>` is valid | `groupId` and `artifactId` are separate fields — `groupId` = publisher (reverse-domain, avoids naming collisions), `artifactId` = the specific library. Together they're a unique coordinate, like a folder path + filename. |
| `ddl-auto=update` keeps the DB schema in sync live, as code changes | Runs **once, at startup** only, comparing entities to the schema. It's also **additive-only** — renaming an `@Entity` field creates a new orphaned column, it never renames the old one. Real renames need Flyway/Liquibase. |
| A field named `text` can map to a column named `name` | Field names must match column names exactly, unless overridden with `@Column(name = "...")` — a mismatch doesn't error immediately, it just silently manages two different things. |
| `@GeneratedValue` alone is enough for an auto-increment `SERIAL` column | Needs an explicit `strategy = GenerationType.IDENTITY` to match how `SERIAL` actually generates values — otherwise Hibernate's default strategy may not line up. |
| Entity fields must be `private` or Hibernate won't work | Hibernate uses reflection regardless of access modifier — `private` is a Java encapsulation convention worth following, not a technical requirement. |
| A `JpaRepository` interface with zero method bodies can't actually do anything | Spring Data JPA generates a real implementing class for it at runtime (a dynamic proxy over Hibernate), using only the declared generic types (`JpaRepository<Product, Integer>`) — nobody, not even Spring's own authors, wrote that specific implementation by hand. |
| JPA and Hibernate are the same thing | JPA is a specification (`@Entity`, `@Id`, etc. — inert until read via reflection, same as `@SpringBootApplication`). Hibernate is the implementation that actually reads them, generates SQL, runs it via JDBC, and maps rows back to objects. |
| An entity without public getters would still serialize fine to JSON | Jackson relies on public getters to know what to serialize — fields alone aren't enough; without getters the API would return an empty `{}`. |

## Part 7 — Full pipeline through Nginx

Confirmed `http://localhost/api/products` returns the same live,
database-backed JSON as hitting Spring Boot directly on port 8080 — the
complete chain, browser to Postgres and back, through every layer above.

**Confusions resolved**

| ❌ Misconception | ✅ Reality |
|---|---|
| `http://localhost/api/` (no further path) 404-ing means something's broken | Correct behavior — Nginx strips `/api/`, forwards `/` to Spring Boot, and no controller has a `@GetMapping("/")`. 404 is the expected outcome here, not a sign anything failed. |

---

## What this ended up demonstrating, end to end

```
Browser
  │  GET :80
  ▼
Nginx  ──── location /      → reads server/index.html, style.css, script.js from disk
  │
  └──── location /api/ ──── strips prefix ──── proxy_pass ────┐
                                                                ▼
                                              Spring Boot process (:8080)
                                                Tomcat (embedded, HTTP + servlet lifecycle)
                                                  → DispatcherServlet (routes by @GetMapping)
                                                    → @RestController (HelloController / ProductController)
                                                      → JpaRepository (Spring-generated implementation)
                                                        → Hibernate (JPA implementation, generates SQL)
                                                          → JDBC driver
                                                            → Postgres (via DBngin, :5432)
```

Every arrow in that diagram was, at some point during this exercise, the
thing that *didn't* work on the first attempt — which is a large part of why
each one is now understood rather than just memorized.
