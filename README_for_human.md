# Building new Agents and Skill/tools integrations for Hans

This document provides an overview of how to build agents to solve business problems locally and how to integrate them into our production agent, Hans. You will require Claude Code to create the first set of skills and scripts (code) for your agent to use, in addition to the agent itself.

### Create the foundation - build a skill solving a real problem *and prove it!*

This is the most demanding part in terms of effort/work, but with a very gentle learning curve. You will need to build a combination of skill(s) and code(s) capable of replicating results over and over with the necessary guardrails so everyone can use it. If it can only be run successfully by you, then the skill you have built is not replicable.

1. Use Claude Code’s skill creation skill to create a new skill (already integrated in Claude). Prompt it to create a new skill and explain what you want to achieve.
2. Iterate on the skill, think of the possibilities of inputs the skill should expect, and the desired outputs.
   1. This takes multiple “rounds”; I highly recommend iterating as you build “real stuff” (if possible). You avoid duplicating work, and you learn and build the skill by doing real work, not hypothetical work.
3. Standardize and make repetitive work programmatic. Think of the parts of the job that actually require an agent to work/reason on them and the parts that don’t. I.e. If you want the output always in the same format, standardize the format into some code (or HTML structure, Markdown, etc) so the agent is not building the same format over and over from scratch; it is inconsistent and burns a lot of tokens (takes more time and $).
4. You want to hit a point of consistently good results. When that happens, you are good to go.

### Confirm it works “elsewhere” - Install Docker

Once your skill(s)+tool(s) work, it is time to test on a “clean slate”. Files on your desktop, previous conversation history, among other things, might taint the way Claude Code works. *I.e. You asked for something to be blue, then Claude started making it blue every time.* The problem here is whether Claude Code is fabricating that preference from your conversations OR if it is actually in the skill and tooling. Therefore, we need Docker to get a free sandbox environment to do tests on.

1. Go to [https://docs.docker.com/desktop/setup/install/mac-install/](https://docs.docker.com/desktop/setup/install/mac-install/) to install Docker.
2. Get Docker Engine running.

### Install NanoClaw

Get NanoClaw installed and set up to run your agent inside a container.

1. As a prerequisite, you will need to install claude-code. This assumes you have the Claude desktop app.
   1. Open your terminal and run:

      ```bash
      curl -fsSL https://claude.ai/install.sh | bash
      ```

   2. Then restart your terminal by running:

      ```bash
      source ~/.zshrc
      ```

2. Download the GitHub repository of NanoClaw: [nanocoai/nanoclaw](https://github.com/nanocoai/nanoclaw)
   1. Or in the terminal:

      ```bash
      git clone https://github.com/nanocoai/nanoclaw.git nanoclaw-v2
      cd nanoclaw-v2 && bash nanoclaw.sh
      ```

   Answer the following questions in your terminal for the agent setup:

   - **How would you like to begin?** → `Standard setup`
   - **How should we create your first agent?** → `Fresh agent`
   - **Where should your assistant's sandbox image come from?** → `Build it here`
   - **How would you like to connect to Claude?** → `Sign in with my Claude subscription`

3. After the setup (questions) and automatic run. Confirm the credential has landed:

   ```bash
   pnpm exec tsx setup/index.ts --step auth -- --check
   ```

   You should expect the outcome `SECRET_PRESENT: true`

### Run the test

Once everything is set up, you are ready to do the test. You will need to download Hans's repository zip file and use it to “run hans” internally with your new skill mounted.

1. Download Hans's repository zip from [TheLumos/nanoclaw-hans](https://github.com/TheLumos/nanoclaw-hans) (you will need a GitHub account with access to Lumos repositories).
   1. Extract the zip on your computer.
2. Download [SETUP-NOTES.md](SETUP-NOTES.md), and place them into your skill(s) and script(s) folder.
3. For the next prompt, you will need to include the following folders. And within the prompt, give a name to your agent to “differentiate” from “Hans” and the job this agent will do: a one-sentence description of what your skill does.
   1. Where you extracted Hans's zip.
   2. The folder with your skill(s) and script(s)
   3. Add the nanoclaw-v2 install folder.

Prompt:

```text
NanoClaw is installed in the nanoclaw-v2 folder. I want a new local agent.

- Name: <Otto>
- Skill folder: <folder name>  — my scripts and skill docs are in here
- The job: <one or two sentences>

Build its persona from Hans's in the nanoclaw-hans folder: keep his operational
discipline, guardrails and tone verbatim, and rewrite only the opening identity
paragraph to the new job. Do not install Hans's production skills (Mongo, GCS,
PostHog, Vercel, Slack formatting) — this agent gets no production credentials.

Read everything in my skill folder first, then mount it into the agent's container
and write one thin skill pointing at it.

Read SETUP-NOTES.md in my skill folder before you start.

Tell me which files you'll create or change before you change them.
```
