# Building a "My Library Account" app with an AI coding assistant

This repository contains no code. It holds the two text files that were used to build a
patron-facing Alma account app with Claude Code, so that other institutions can reproduce
the build with their own identity provider and Alma instance.

| File        | What it is                                                           | When it is read                        |
|-------------|----------------------------------------------------------------------|----------------------------------------|
| `PROMPT.md` | The specification: purpose, stack, authentication, Alma integration, functional requirements, acceptance tests, and known pitfalls | Once, when you ask the assistant to build the app |
| `CLAUDE.md` | Project instructions for the assistant: commands, layout, and rules that must hold during later changes | Automatically, at the start of every Claude Code session in the project |

The app itself is a Spring Boot backend that signs patrons in with Amazon Cognito, calls the
Alma Users API with a server-side key, and serves a React + Material UI frontend with three
tables: active requests, loans, and fees.

## How to use these files

1. **Create an empty project directory** and copy both files into it.

2. **Adjust the spec for your institution.** Open `PROMPT.md` and check:
   - Section 3 if you don't use Cognito. Section 9 has drop-in replacements for generic
     OIDC providers, social logins, and SAML.
   - The default claim name for the Alma identifier (`custom:almaUser`). Use whichever claim
     your identity provider puts the Alma primary ID in.
   - The package manager line in section 2 if your team uses npm rather than pnpm.
   - Section 8 if you want more than read-only access.

3. **Start the assistant in that directory and give it the spec.** With Claude Code:

   ```
   claude
   > Build the application described in PROMPT.md.
   ```

   The assistant reads `CLAUDE.md` on its own. It will scaffold the backend and frontend,
   write the tests listed in section 7, and produce a README with the configuration steps.

4. **Configure and run.** The generated project expects environment variables for the Alma
   base URL and API key and for the identity provider. Register the callback URL
   `http://localhost:8080/login/oauth2/code/<registration>` and the sign-out URL on your
   identity provider's app client before the first login.

5. **Keep `CLAUDE.md` with the code.** It is the file that stops later changes from reintroducing
   the problems in section 10 of the spec. Edit it as your conventions evolve.

## Why two files

A spec is read once and can be long. Instructions that must hold for every future change
are read every session and should be short. Mixing them means either a bloated per-session
file or a spec that gets ignored. The split is the same one used by spec-driven development
tools, where requirements live in one document and standing rules in another.

## What to expect

The original build took one session for the working app and a second session for
configuration debugging. Every issue in that second session was configuration rather than
code, and each one is now recorded in section 10 of `PROMPT.md` so the assistant can avoid
it. Expect the assistant to ask nothing and to deliver a project that compiles and passes its
own tests; the remaining work is supplying real credentials and identity provider settings.

## Adapting beyond Alma

The structure transfers to any "show me my data from system X behind our login" app. Replace
section 4 with your system's API and section 5 with the tables you want. Sections 3, 6, 7,
and 9 can stay largely as they are.

## Using other assistants

`CLAUDE.md` is the filename Claude Code reads. Cursor reads `.cursor/rules`, GitHub Copilot
reads `.github/copilot-instructions.md`, and several tools read `AGENTS.md`. The content is
plain Markdown, so copy or symlink it under the name your tool expects.
