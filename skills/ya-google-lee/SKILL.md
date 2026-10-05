---
name: ya-google-lee
description: Kill Google API dependencies for your app. Use when asked to de-Google, unbundle, self-host, make provider-agnostic, or take an AI Studio / Firebase Studio export and make it run standalone.
---

# Ya-Google-Lee

Taking an app built on Google's stack and making it run without it.

This is written from doing it once, to a real app: a Gemini + Firebase
creative suite exported from Google AI Studio, ported so it runs on local models
or on whatever key its user has. The shape below is what that port actually
looked like, including the parts that went wrong.

## The rule that decides everything else

**Replace the transport, keep the call sites.**

The temptation is to rewrite each studio as you reach it, because each one
looks slightly wrong once you are inside it. Do not. Keep every exported
function's name and signature identical to the Google original, change only
what happens inside it, and the rest of the app keeps compiling while you
work. In that port `services/geminiService.ts` became
`services/providerService.ts` with the same 25 exports —
`generateCreativeImage`, `generateLogoImages`, `generateText`,
`generateJSON`, `editLogoImage` and the rest — and not one of the fifteen
component files needed editing to build.

The corollary: a function whose behaviour genuinely cannot survive the port
still keeps its name, and **throws with an explanation**. See Veo, below.

## What to find first

Grep for these before planning anything. The count tells you the size of the
job better than the line count does:

```bash
grep -rn "@google/genai\|GoogleGenAI\|generativelanguage" --include=*.ts --include=*.tsx .
grep -rn "process\.env\.API_KEY\|process\.env\.GEMINI_API_KEY" .
grep -rln "firebase\|firestore\|FirebaseProvider" --include=*.ts --include=*.tsx .
grep -rn "drive\.googleapis\|googleapis\.com" .
grep -rn "veo\|Veo" --include=*.ts --include=*.tsx .
cat metadata.json          # AI Studio names its capabilities here
cat firebase-applet-config.json firestore.rules 2>/dev/null
```

`metadata.json` is the fastest read: `MAJOR_CAPABILITY_SERVER_SIDE_GEMINI_API`
means the key never existed in the client and Google was proxying, so there is
no key to migrate — there is a call to *build*.

## The four bindings, in the order to cut them

### 1. The model call — always first

Everything else depends on this and nothing depends on it. In the original,
the renderer holds `new GoogleGenAI({apiKey: process.env.API_KEY})` and calls
it directly. Two problems: the key is in the client bundle, and the provider
is hard-coded.

Move the call behind a boundary the app already has — in Electron that is IPC
to main, on the web it is your own server route — and route by capability, not
by vendor:

```
text / JSON      → the local server if one is running (Ollama, LM Studio,
                   llama.cpp), else the chosen hosted provider
images           → the local diffusion pipeline, else a hosted image model
grounded search  → Gemini only; nothing local does live web grounding
video            → see Veo
```

**The routing rule that matters**, learned by getting it wrong first: *the
model the user picked is the model that answers.* The first version was
"local, with a hosted fallback", which meant the thing that produced your work
depended on which server happened to be up that afternoon. Two runs of the
same prompt came back from two different models with no indication. List
everything reachable in one picker — local servers, OpenRouter, OpenAI,
Anthropic, NVIDIA NIM, Google included — and make the choice explicit.

Keep Google in the list. De-Googling is about removing the *requirement*, not
the option; an app that cannot use Gemini when its user has a Gemini key is
worse than the one you started with.

### 2. JSON responses — the one that breaks quietly

`response_format`/`responseSchema` is honoured by Gemini and by OpenAI, and
*variably* by every local server. A local model will happily wrap valid JSON
in a sentence or a ```json fence. `JSON.parse` on the raw string then throws,
and it throws in a studio three screens away from the thing you changed.

Slice before parsing, in the one helper every caller already goes through:

```ts
export async function generateJSON<T>(system: string, user: string): Promise<T> {
  const raw = await callModel(system, user, { json: true });
  const start = raw.indexOf('{');
  const end = raw.lastIndexOf('}');
  return JSON.parse(start >= 0 && end > start ? raw.slice(start, end + 1) : raw) as T;
}
```

### 3. Firebase — usually deletable, not portable

Look at what it is actually doing. In that port it was auth plus a document
store for prototypes and design references, and the app was becoming
single-user and local — so auth went away entirely and the documents became
local state. That is the common case for a tool: Firebase was there because
AI Studio put it there, not because the app needs a multi-tenant database.

If it genuinely is multi-user, swap the client, do not keep the config. And
read `firestore.rules` before you throw it away — it is the clearest statement
of the data model anyone wrote down.

**On the `apiKey` in `firebase-applet-config.json`:** it is a Firebase *web*
key. It identifies the project and authorises nothing; it is designed to ship
in client code. The security is in `firestore.rules`. Do not treat finding it
as a leak, and do not paste it into a different project either.

**Never make Firebase all-or-nothing.** Whichever way the call above goes —
deleted, swapped, or kept — the port is not done until Firebase is a choice
the app's own user makes, not one baked in at build time. Two failure modes,
both seen in practice:

- **A module that reaches Firebase just by being imported.** A file that
  calls `initializeApp()` at the top level, or self-invokes a "test the
  connection" check on load, hits the network before anything decided
  whether cloud sync should even be on. That happens whether or not any
  caller ever uses the module.
- **An app that cannot run at all without a working Firebase project**,
  even when nothing about the app is inherently multi-user.

Fix it the same way the Google key itself is handled: a runtime toggle,
default OFF, read fresh on each check (not a build-time flavour flag — this
is a preference, not a bundle split). Gate `initializeApp`/`getAuth`/
`getFirestore` behind it, and keep every existing export's name and call
shape (`db`, `signInWithGoogle`, ...) so nothing importing them has to
change — a `Proxy` that lazily initialises on first real property access and
throws a plain-language error when the toggle is off is enough: `db` stays
`db`, `collection(db, 'gallery')` still compiles, and every call site that
already wraps Firestore calls in try/catch (most AI Studio exports do,
because Firestore already fails unpredictably) catches the "cloud sync is
off" error exactly where it would have caught a real Firestore error. Put
the toggle itself in Settings, not just an env var — the app's own user
should be able to flip it without a rebuild.

**If you have your own backend, prefer it over a toggle.** Firebase in these
apps is rarely doing more than user identity (auth) and a small per-user JSON
store (a gallery collection, a settings doc). Once a replacement exists on a
backend you control, remove `firebase.ts`, the `firebase` package dependency and
the `*-applet-config.json` file entirely rather than leaving a disabled path
behind. Point the app at your own auth and storage routes instead of standing up
a new service, and test writes against a local or staging copy of that backend,
never the production database.

### 4. Drive, and the rest of the SDK surface

File pickers, Drive storage, Google auth status widgets. These are almost
always a thin veneer over "open a file" and "save a file". Replace with the
platform's own file dialog. Delete the status widget rather than porting it —
a green "Google Cloud ✓" pill in an app that no longer uses Google is worse
than no pill.

## What you cannot port, and what to do about it

Some capabilities have no local equivalent today. As of writing that is
text-to-video (Veo), live web-grounded search, and anything using a model
nobody has released weights for.

Do not silently stub these. A button that appears to work and never returns is
the worst outcome of a port. Keep the export, throw, and make the error the
documentation:

```ts
const NO_LOCAL_VIDEO =
  'Veo is a hosted Google model and there is no local text-to-video pipeline here yet. '
  + 'Add a Google key in Settings and unlock Google to use it, or use the Motion library '
  + 'for animation you can render yourself.';

export async function generateVeoVideo(): Promise<never> { throw new Error(NO_LOCAL_VIDEO); }
```

Three sentences: what it is, why it cannot run, what to do instead.

## Order of work

1. Inventory with the greps above. Count call sites per binding.
2. Build the new service module with the **same export names**. Point every
   function at the new transport. Nothing else changes yet.
3. Swap the import in one component. Build. Fix what breaks there only.
4. Swap the rest of the imports at once — if step 3 was clean, they are clean.
5. Delete Firebase, Drive and the status widgets last, when nothing imports
   them. Deleting first turns one port into fifteen broken files.
6. Run it with **no key configured at all**. This is the test most ports skip
   and it is the one that matters: every failure should name what is missing
   and where to set it, not surface as a stack trace or an empty pane.

## Rules

- **Never enter the user's API key** into a file, a form or an env var.
  Provide the field and the instructions; the key is theirs to type.
- **Keep both builds runnable.** The Google build is the reference. When the
  port produces different output, being able to run the original is how you
  find out which one is wrong.
- **`process.env.API_KEY` in a Vite app is inlined into the bundle**, via
  `define` in `vite.config.ts`. Anyone who downloads the app has the key. Say
  so plainly if the app being ported shipped that way.
- **Check the licence before re-releasing** an app you did not write.
- A port is finished when the app runs with no Google dependency reachable —
  not when it compiles.
