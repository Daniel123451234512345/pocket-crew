# Pocket Crew

**A mixed AI crew in my group chat. One AI plans, one builds, a different AI checks it, and my friends and I are the bosses. No terminals.**

Built for the [Flip the Switch Hackathon](https://www.wemakedevs.org/hackathons/switch) (Track 02: Best Personal or Creative Workflow).

🎥 **Demo video:** _link here_
🌐 **Live site the crew built and shipped:** https://daniel123451234512345.github.io/pocket-crew-demo/ ([repo](https://github.com/Daniel123451234512345/pocket-crew-demo))

---

## The problem

I'm a first-year student with too many side projects and not enough time at my laptop. AI coding agents are great, but they live in a terminal on my MacBook. When I'm on the bus and think "my site needs dark mode," I can't do anything about it.

Also, a single agent checking its own work isn't real review. It shares its own blind spots. And a one-line message from my phone ("make my portfolio look legit") gets a bare-minimum result, because nobody turns it into a real plan.

## What Pocket Crew does

Every side project gets its own chat room. In each room there's a three-agent crew:

| Agent | Runs on | Job |
|---|---|---|
| **planner** | Claude Code | Turns a short, vague request into `PLAN.md`: what to build, how it should look, and a "done when" checklist. Never asks questions first; it lists its assumptions instead. |
| **builder** | Claude Code | Builds `PLAN.md` to a show-off standard, fixes bugs, ships when a human approves. |
| **tester** | OpenCode | Checks the work against `PLAN.md`, runs it, and does a design check at phone and desktop width. Reports bugs. **Never edits code.** |

And there are humans: me, plus anyone I invite. We're the bosses.

```
 me (phone):  "@planner make my portfolio look legit"
     planner:  writes PLAN.md (sections, style, done-when, assumptions)
               → "@builder please build PLAN.md"
     builder:  builds it → "@tester please check it against PLAN.md"
      tester:  "@builder 1. projects grid scrolls sideways at 375px ..."
     builder:  fixes it → "@tester re-check please"
      tester:  "@builder passes ✅"
     builder:  "Ready to ship: portfolio refresh. Approve?"
 me (phone):  "@builder ship it"
     builder:  git commit + push → "Shipped 🚀 https://…github.io/…"
```

Nothing goes live until a human says **ship it**. For small tweaks ("make the button blue") I skip the planner and message the builder directly.

## How Switch is used (and why this doesn't work without it)

[Switch](https://docs.switchagents.ai) connects AI agents and people in the chat apps they already use. In Pocket Crew it's the central piece:

1. **Switch runs all three agents as members of a chat room.** planner and builder (Claude Code) and tester (OpenCode) are started and managed by the Switch Console on my Mac and show up as bot users in Mattermost. I don't open a terminal.
2. **Switch is the only link between the agents.** Each agent is its own separate session, and Claude Code and OpenCode are separate tools from different companies with no way to talk to each other. Every handoff (planner → builder → tester → builder) is an @mention that travels through the Switch room.
3. **Switch gives each room its own agent session.** The same builder in `#portfolio` and `#stats-project` keeps separate context, so each project has its own memory.
4. **Switch puts people in the loop.** Anyone in the room can give orders or approve. The approval gate ("ship it") is a chat message from a person.

### "Why not just use Claude Code?"

Fair question. Claude Code can build *and* test with its own subagents. What it can't do:

- **Bring in a different AI as reviewer.** Claude testing Claude is grading its own homework. Here the tester is a different tool on a different model. Only Switch lets the two work as one crew.
- **Work with other people.** Claude Code is one person talking to one session. A Switch room is a group chat: my friend can join `#portfolio`, ask for a feature, and approve the deploy.
- **Live where I already am.** The crew runs in a chat app (Mattermost here, and the same rooms can be bridged to Discord or Slack), with a room per project, instead of a terminal window per project.

**Delete Switch and Pocket Crew stops existing:** the agents go back to separate terminals, can't hand work to each other, and nobody can reach them from chat.

## Architecture

```
  ┌──────────── Mattermost (chat app) ────────────┐
  │  #project-demo     #portfolio     #…          │
  │   me, friend, @planner, @builder, @tester     │
  └──────────────────────┬────────────────────────┘
                         │  Switch collaboration bridge
               ┌─────────┴─────────┐
               │   Switch server   │  rooms, sessions per room, routing @mentions
               └─────────┬─────────┘
                         │  Switch agent controller (my Mac)
        ┌────────────────┼────────────────┐
  planner (Claude)  builder (Claude)  tester (OpenCode)
        └──── same project folder (git repo + PLAN.md) ────┘
                         │ "ship it" → git push
                    GitHub Pages (live site)
```

## Set it up yourself

**You need:** macOS/Linux, [Docker Desktop](https://www.docker.com/products/docker-desktop/), [Switch Console](https://docs.switchagents.ai), [Claude Code](https://claude.com/claude-code), [OpenCode](https://opencode.ai), the [GitHub CLI](https://cli.github.com) (`gh`), and git.

1. **Start Docker Desktop first.** The local Switch server runs Mattermost in Docker. If Docker is off, the room's bridge crashes with "Bridge not running".
2. **Open Switch Console** and start a **Local Switch server**. This also starts Mattermost.
3. **Make a project folder that's a GitHub repo with Pages on:**
   ```bash
   mkdir my-site && cd my-site && git init -b main && echo '<h1>hi</h1>' > index.html
   git add -A && git commit -m init
   gh repo create my-site --public --source=. --push
   gh api -X POST repos/<you>/my-site/pages -f "source[branch]=main" -f "source[path]=/"
   ```
4. **Create the crew from the template:** in Switch Console → Templates, import [`templates/pocket-crew.template.yaml`](templates/pocket-crew.template.yaml), set `project` (e.g. `my-site`) and `directory` (your folder from step 3), and deploy. This creates `my-site-planner`, `my-site-builder`, `my-site-tester` and the `#my-site` room.
   - *Manual alternative:* create three agents in Switch Console (`planner` and `builder` on Claude Code, `tester` on OpenCode, all with "This computer" and your folder as the working directory), paste [`agents/planner.md`](agents/planner.md), [`agents/builder.md`](agents/builder.md) and [`agents/tester.md`](agents/tester.md) as their instructions, then create a room with all three. Their names just need to end in `planner`, `builder` and `tester`.
5. **Allow Switch Console access to your folder:** on macOS, grant it access under System Settings → Privacy & Security if the agents can't read or write files.
6. In the room, try:
   ```
   @my-site-planner turn this into a portfolio site for a college student who loves data science
   ```
   Wait for "Approve?", reply **@my-site-builder ship it**, and open your GitHub Pages link.

**Another project?** Deploy the template again with a new `project` name. You get a new room with its own separate memory.

### Gotchas I hit (so you don't have to)

- **Open Docker before Switch Console.** If the bridge crashed, quit the Console, stop the leftover server (`pkill -f switch_core.main`), then open the Console once and let it start everything.
- **Don't quit Switch Console while agents are working.** Quitting stops every agent mid-task. Closing the window is fine.
- **Agents working in the wrong folder?** Set the working directory in the agent's settings, restart the agent, and start a fresh chat.
- **Always @mention the agent you're talking to.** Switch only delivers a message to an agent that's addressed, so approve with `@builder ship it`; a bare "ship it" never reaches builder.
- **Only @mention the agent you want to start.** If your request also mentions the tester, it wakes up before there's anything to test. The agents hand off to each other by themselves.
- **OpenCode with no login** uses a free built-in model. It works but is slow (2–4 min per check). `opencode auth login` with your own key speeds it up.

## Repo contents

```
README.md                            this file
agents/planner.md                    planner's instructions
agents/builder.md                    builder's instructions
agents/tester.md                     tester's instructions
templates/pocket-crew.template.yaml  one-click Switch template: 3 agents + room
```

## Built with

Switch (SandboxAQ / Flint AI) · Mattermost · Claude Code · OpenCode · GitHub Pages

Made solo by Daniel Ho (UT Austin, Statistics & Data Science) during the hackathon, Nov 4–7, 2026.
