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

- **PowerShell won't run an executable from the current directory by name
  alone.** `nginx.exe -s reload` failed with "not recognized" — not because
  Nginx was missing, but because PowerShell (unlike `cmd.exe`) doesn't search
  the current directory automatically. Fix: `.\nginx.exe -s reload`.
- **Two `nginx.exe` processes in Task Manager is correct, not a bug.** Nginx
  runs a **master process** (reads config, manages workers, handles signals
  like `-s reload`) and at least one **worker process** (actually accepts
  connections and serves requests). Seeing exactly one of each is the healthy
  state.
- **Why bother with Nginx at all, when Spring Boot has an embedded Tomcat
  that's already a complete HTTP server?** Because Tomcat *can* serve static
  files, but every request — even a static image — still passes through the
  full servlet lifecycle machinery, which is wasted overhead for something
  as simple as "read a file, write bytes." Nginx does that specific job with
  far less overhead, and in production also serves as the single public
  entry point, TLS terminator, and load balancer in front of one or more app
  server instances.

## Part 2 — Serve a static site through Nginx

Wrote `index.html`, `style.css`, `script.js` by hand and pointed Nginx's
`location / { root ...; index index.html; }` at that folder.

**Confusions resolved**

- **Windows hides file extensions by default.** Files saved from a plain
  text editor ended up as `index.html.txt` without any visual indicator.
  Fixed via File Explorer → View → show file name extensions.
- **HTML syntax slips that looked plausible but aren't valid**: `<head/>` as
  a self-closing tag (not how HTML closes an element — needs `</head>`);
  visible content placed inside `<head>` instead of `<body>` (`<head>` is for
  metadata, not what the reader sees); `<script ref="...">` and
  `<link ref="...">` (the real attribute is `src` for `<script>`, `href` for
  `<link>` — `ref` isn't a real HTML attribute at all); `<script ...>`
  written as self-closing (`<script src="..." />`) — script tags always need
  an explicit closing tag since they can contain inline code between them.
- **CSS needs a semicolon after every declaration, including the last one in
  a block.** It's easy to miss on the final line since the block still
  "looks" closed without it — but it's still invalid and worth fixing on
  sight, not just when it happens to break something.
- **`16sp` isn't a CSS unit** — that's Android's "scalable pixels." The CSS
  equivalents are `px`, `em`, `rem`.
- **JavaScript function syntax borrowed from other languages doesn't
  transfer.** `func updateP() { ... }` (Swift/Go-style) isn't valid — the
  keyword is `function`. Similarly, `p: { onchange = Text("...") }` isn't
  real JS at all — the actual pattern is: select the element
  (`document.querySelector("p")`), then assign directly to a property
  (`.textContent = "..."`), no wrapper function needed to set a string.
  Event handler properties are also case-sensitive and lowercase:
  `onclick`, not `onClick`.
- **Editing a file and refreshing the browser only works if the save
  actually happened.** Several rounds of "I fixed it" showed no change on
  re-check because the file on disk was untouched — a reminder to verify a
  save landed rather than assuming it did.

## Part 3 — A Spring Boot "Hello World"

Generated a project via Spring Initializr (Maven, Java, `Spring Web` only),
wrote one `@RestController` with a `GET /hello` endpoint returning a plain
string, ran it, and hit `http://localhost:8080/hello` directly.

**Confusions resolved**

- **`@RestController` isn't "conforming to a REST protocol."** REST isn't a
  protocol at all — it's an architectural style built on top of HTTP, which
  *is* the actual protocol. What `@RestController` really does is combine
  `@Controller` (marks the class as a request handler) with `@ResponseBody`
  (whatever a method returns gets written directly into the HTTP response
  body, instead of being treated as the name of an HTML view template to
  render). Plain `@Controller` is for server-rendered pages via a template
  engine — a legitimate but different architecture, not a "worse" version of
  `@RestController`.
- **Initializr's Java version dropdown won't always list your exact
  installed version.** With JDK 23 installed, the practical rule is: pick
  the highest listed option that's *less than or equal to* your installed
  version (21, in this case) — a newer JDK can run code targeting an older
  version, not the reverse.
- **`mvn spring-boot:run` failing with "not recognized"** usually means
  Maven isn't installed globally — but Spring Initializr projects ship with
  a **Maven Wrapper** (`mvnw` / `mvnw.cmd`) specifically so a global Maven
  install isn't required. `.\mvnw.cmd spring-boot:run` works without one.
- **Testing in the wrong browser produced a false "it's broken" signal** —
  an embedded/sandboxed browser rendered nothing, while a real browser (Edge)
  showed the response correctly. Worth ruling out the test environment
  itself before assuming the app is broken.

## Part 4 — Reverse proxy Nginx → Spring Boot

Added a second `location` block to the same Nginx `server {}`, using
`proxy_pass` to forward matching requests to the Spring Boot app on port
8080.

**Confusions resolved** — this was the deepest rabbit hole, and worth
recording in full because the mechanism isn't obvious from the directive
names alone.

- **`location /` already had a job (serving static files via `root`), so a
  second, distinct path prefix (`/api/`) is needed** for the proxy — not
  because `proxy_pass` can't target port 8080 from `location /` (it can,
  from any location block), but because one `location` block can't
  simultaneously serve files *and* proxy for the same path without
  conflicting.
- **The trailing slash on both `location` and `proxy_pass` isn't
  decorative — it changes Nginx's behavior entirely.** The actual mechanism,
  worked out by testing every combination directly rather than taking the
  rule on faith:
  - `location /api/ { proxy_pass http://localhost:8080/; }` — Nginx matches
    the `/api/` prefix, **strips it**, and appends whatever's left onto the
    path portion of `proxy_pass` (here, `/`). So `/api/hello` → strip
    `/api/` → remainder `hello` → append onto `/` → forwarded as `/hello`,
    which matches the controller's `@GetMapping("/hello")`.
  - Drop the trailing slash from `proxy_pass` only
    (`http://localhost:8080;` — no path component at all): Nginx has
    nothing to append the stripped remainder *onto*, so it stops
    rewriting entirely and forwards the **original, full path unchanged** —
    `/api/hello` arrives at Spring Boot as literally `/api/hello`, which
    doesn't match any mapping → 404 (Spring Boot's own Whitelabel Error
    Page, not an Nginx error — proof the proxy hop itself worked, just to
    the wrong path).
  - Drop the trailing slash from `location` only (`location /api {`) with a
    slashed `proxy_pass`: for a request to exactly `/api`, the match is
    exact and the remainder is empty, so it still works, forwarding `/`
    correctly. But for `/api/hello`, the matched prefix is only `/api` and
    the remainder becomes `/hello` — **including its own leading slash** —
    which then gets appended onto `proxy_pass`'s `/`, producing `//hello`
    (a double slash). It's a subtle, inconsistent bug that only shows up on
    sub-paths, not the one you're likely to test first.
  - **The fix is keeping the trailing slash consistent on both sides** —
    the only combination that behaves predictably for every possible
    sub-path, not just one you happened to test.
- **The `/` in `proxy_pass http://localhost:8080/;` is doing real work, not
  padding.** It's the explicit path Nginx substitutes the stripped prefix
  onto. Without any path at all in the URL (not even a bare `/`), there's no
  anchor for the substitution, which is exactly why omitting it disables
  prefix-stripping altogether rather than just "defaulting to root."
- **Nginx config changes are inert until reloaded.** A running worker holds
  its configuration already parsed in memory — editing `nginx.conf` on disk
  changes nothing about the live process until `nginx -s reload` explicitly
  tells the master process to re-read the file and start new workers with
  it. `reload` (vs. `stop`) is specifically a **graceful** operation: old
  workers finish in-flight requests before exiting, new workers (with the
  updated config) start accepting new ones immediately — zero dropped
  connections, the same principle behind zero-downtime rolling deploys
  elsewhere in backend infrastructure.
- **Tomcat's role didn't change by adding Nginx in front of it.** Tomcat is
  inherently both a web server (speaks HTTP) and an application server (runs
  servlet/Java code) — that's true with or without Nginx. What changed is
  traffic *routing*: static-asset requests now never reach Tomcat at all,
  while dynamic requests still do, exactly as they always could. It's fair
  to describe Nginx as "the web server" and Tomcat as "the application
  server" *for this specific architecture's division of labor* — but that's
  a statement about how the two are being used here, not a fact about what
  either is inherently limited to.
- **The loopback address, and why it survives even with no network
  connection.** `localhost` resolves to `127.0.0.1`, a special address
  meaning "this same machine" — traffic to it is handled entirely inside the
  OS's own networking stack and never touches a physical network adapter, so
  it keeps working with WiFi off or no cable plugged in. Source and
  destination being the same machine is the literal meaning of "loopback":
  the data never leaves to loop back from anywhere external.
- **Client and server are roles, not fixed identities.** The same Spring
  Boot process is a *server* to the browser (accepting requests on 8080/via
  Nginx) and, once Part 6 adds a database call, simultaneously a *client* to
  Postgres (initiating a connection outward to port 5432). Long backend
  chains are just this same relationship repeated at each hop.
- **A client doesn't need a fixed, known port the way a server does.** A
  server (Postgres, Nginx, Tomcat) *listens* on a specific, well-known port
  by design, so others can find it. A client (TablePlus, or Spring Boot
  acting as a DB client) only gets a disposable, OS-assigned port for the
  duration of one connection — nobody needs to know or remember it, since
  the client is the one initiating contact, not waiting to be found.

## Part 5 — Local database via DBngin + TablePlus

Installed DBngin, spun up a local Postgres instance (host `127.0.0.1`, port
`5432`, user `postgres`, no password), installed TablePlus, connected to it,
and created a `products` table (`id`, `name`, `price`) with two rows inserted
by hand.

**Confusions resolved**

- **DBngin doesn't bundle database engines — it downloads them on first
  use.** Tapping "start" on a fresh instance kicks off a download of the
  actual Postgres binaries before anything can run; this is expected, not a
  stall or an error.
- **A running database has no UI of its own to look at.** Postgres speaks
  its own wire protocol over its port and renders nothing visually — same
  pattern as Nginx having no window either. TablePlus is the **client** that
  connects to that port, speaks the protocol, and renders the response as a
  visual grid. "The database instance is running but I see nothing" is the
  expected state without a client attached.
- **DBngin (the manager) and Postgres/MySQL (actual databases) are different
  layers.** DBngin doesn't store or query data itself — it's a GUI for
  downloading, starting, and stopping the real engines. Postgres and MySQL
  are peer, competing implementations of the same category of software
  (relational database management systems); DBngin can launch either.
- **TablePlus's GUI table-builder can silently leave changes unsaved.**
  Added `name` and `price` columns showed in a different color (unsaved/
  pending) than the `id` column (already persisted) — and a
  subsequent `INSERT` failed with `column "name" ... does not exist`,
  because the GUI had staged the columns but never actually committed them
  to Postgres. Resolved by dropping the incomplete table and recreating it
  in one atomic `CREATE TABLE` statement via SQL instead of the GUI wizard —
  which also sidesteps the ambiguity entirely, since the GUI wizard is just
  a friendlier front-end generating the same SQL underneath.
- **A schema isn't something you separately "find" or create** — Postgres
  creates the default `public` schema automatically with every new database;
  TablePlus's sidebar sometimes collapses/hides that layer visually when
  there's only the one default schema, showing "Tables" directly instead.

## Part 6 — Connect Spring Boot to the database

Added `spring-boot-starter-data-jpa` and the `org.postgresql:postgresql`
driver to `pom.xml`, configured `application.properties`, wrote a `Product`
`@Entity` matching the existing table, an empty `ProductRepository`
interface extending `JpaRepository`, and a `GET /products` endpoint that
calls `.findAll()`.

**Confusions resolved**

- **A Maven dependency's `groupId` and `artifactId` are two separate
  fields, not one combined string.** First attempt wrote the whole
  `org.postgresql:postgresql` coordinate into a single `<artifactId>` tag.
  `groupId` identifies the publisher/organization (reverse-domain style,
  same reasoning as Java package naming — avoids collisions between
  different publishers using similar names); `artifactId` is the specific
  library that publisher released. Together they're a unique coordinate for
  one library, the same way a full file path needs both a folder and a
  filename to be unambiguous.
- **`spring.jpa.hibernate.ddl-auto=update` only runs once, at application
  startup** — comparing `@Entity` classes against the actual database schema
  and adding whatever's missing. It is **not** a live watcher reacting to
  code changes as they're saved, and it is **additive-only**: it will not
  rename or drop a column to match a changed entity. Renaming a field in
  `@Entity` after the table already exists doesn't rename the real column —
  it creates a brand-new one alongside the orphaned old one. Real schema
  changes (renames, restructuring) belong to dedicated migration tools like
  Flyway or Liquibase in real projects, not `ddl-auto`.
- **An `@Entity` field's name must match its column's name exactly** unless
  overridden with `@Column(name = "...")` — a field named `text` mapped
  against a table column named `name` doesn't error immediately; it silently
  tries to manage two different things.
- **`@GeneratedValue` needs an explicit strategy to match how the column was
  actually created.** Postgres's `SERIAL` is a database-level
  auto-increment; without `strategy = GenerationType.IDENTITY`, Hibernate's
  default generation approach may not line up with it.
- **JPA/Hibernate entity fields being private isn't enforced by Hibernate
  itself** (it uses reflection regardless of access modifier) — it's a
  standard Java encapsulation convention (private fields, public
  getters/setters) worth following anyway, not a technical requirement.
- **A repository interface with zero method bodies still works because
  Spring Data JPA generates a real implementing class for it at runtime**,
  as a dynamic proxy wired to Hibernate underneath — using nothing but the
  interface's declared generic types (`JpaRepository<Product, Integer>`) to
  know what entity and ID type it's managing. `JpaRepository` itself is also
  just an interface with no bodies — the actual code that runs is written by
  neither the developer nor Spring's own interface author, but generated
  specifically for this interface at startup.
- **Hibernate is the implementation of the JPA specification, not JPA
  itself.** `@Entity`, `@Id`, and friends are inert annotations (same "just
  metadata until something reads it via reflection" principle as
  `@SpringBootApplication`) until Hibernate, at runtime, reads them,
  generates the actual SQL, executes it through the JDBC driver, and maps
  the rows back into Java objects.
- **Missing public getters on an entity would have meant an empty `{}` in
  the JSON response** — Jackson (Spring's default JSON library) relies on
  public getter methods to know what to serialize; fields alone, without
  accessors, aren't enough for the object to actually appear in the API
  response.

## Part 7 — Full pipeline through Nginx

Confirmed `http://localhost/api/products` returns the same live,
database-backed JSON as hitting Spring Boot directly on port 8080 — the
complete chain, browser to Postgres and back, through every layer above.

**Confusions resolved**

- **`http://localhost/api/` (no further path) correctly 404s** — not a bug.
  Nginx strips `/api/`, leaving an empty remainder, forwarded as `/` to
  Spring Boot; since no controller has a `@GetMapping("/")`, a 404 is the
  expected, correct outcome, not a sign anything is broken.

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
