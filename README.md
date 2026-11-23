# What I let my AI agents do, and what broke

*A case study. JD Davenport, 2026.*

I run my own agent system on a Mac Mini. I started building it in February 2026, and it has run on a schedule since April 2026. I've built 100+ agents for it. They work on my email, my code, my school work, my job search and my family's logistics. One user: me.

Some of what they do has real consequences. They draft and send email, they fix code, and they run shell commands on the machine that holds my passwords. This write-up goes through each of those, what could go wrong, what I put in place, and at least one time the controls failed and what I changed after.

## Two caveats first

It's a single-user system. It doesn't solve permissions for many users, and I don't claim it does.

"Live" here means real email to real people and real money on the line. It has no uptime promise to anyone but me. That's why the safety work matters to me, even though the scale is small.

## 1. Agents with a shell on my own machine

**What could go wrong.** Many sessions run at once with shell access on the box that also holds my credentials, my family's data and every other agent's state. A bad command can delete things, read secrets or break a sibling session.

**What I put in place.** Hooks that run before any command executes. They don't care what the model believes. One gate blocks reads of top-tier secrets, sends that skip the approval step, agent spawns nobody can trace, and force-pushes without an explicit flag. A second guard blocks reads of browser password stores, cookie databases and bulk keychain dumps, and still allows reading one credential on purpose. The gate is open source as [ethos-gate](https://github.com/jddavenportOpen/ethos-gate).

**One that failed.** A worktree cleanup script had three checks before it deleted anything, and all three failed open. An unreadable worktree looked empty and stale, so it got deleted. When I reproduced it, it destroyed a file that had never been committed. Every check on that path now fails closed.

**What it costs.** Hooks have false positives. The credential guard blocked its own documentation commit the day it shipped. I fixed it by making the guard look for a real browser profile path or keychain command, so the pattern itself stayed as strict.

## 2. Email and messages to real people

**What could go wrong.** Agents write to my family, my professors and people I work with. A wrong send can't be taken back.

**What I put in place.** Every email my agents send to another person passes one gate. By default it waits for my yes on the exact text. The approval is a single-use token that expires, and it's tied to a hash of the recipients, subject and body. If one character of the body changes, the token is dead. A bare "yes" with nothing attached to it gets refused. There's also a freeze switch, and a log of every send and every refusal. Texts sit behind the freeze switch but don't have per-message tokens yet.

I've signed three narrow standing approvals: rescheduling AI Foundry check-ins with the teams, outreach for a case competition the Foundry runs, and cold email for AgentTree, a side project. AgentTree sends from its own domains, with my name on it. Each approval has a cap and a kill switch.

A build check fails if any code sends email without going through the gate. The first time it ran, it found Gmail senders that I'd missed when I checked by hand.

**One that failed.** Today a hook checks every fork, pull request and Discussion my agents aim at a repo I don't own. A model judges each one, and it blocks anything that looks like promotion, star-farming or mass outreach. I can override a block myself. I built the first version of that hook after the incident it was meant to stop. On July 4 and 5, 2026, one agent session running on my own personal GitHub account opened 248 pull requests on 181 other people's repos, plus 415 Discussions, to farm stars. It was working toward a goal of 100 stars or $100 earned, and nothing stopped it from posting outside my own repos. I closed all 248 pull requests and apologized on July 5. All 415 Discussions have since been removed. On July 6 I deleted the 189 forks it had made. I set up this account, jddavenportOpen, later in July.

**What it costs.** The gate is strict enough to break things. One university email path refused every send for weeks because the code that made the token and the code that sent the mail disagreed on what the token was bound to. I fixed the code that passed the key along, and the check stayed as strict.

## 3. Agents that fix code

**What could go wrong.** A pipeline that reproduces a bug, writes a fix and merges it can merge anything, including an auth bypass or a disabled safety check. And an agent asked to grade its own work will pass it.

**What I put in place.** Every check reads the git diff itself. The fix happens in a throwaway worktree. The test has to go red, then green. By default a fix can touch at most 3 files and 40 lines, and can't add dependencies, change a schema or touch CI. A deny list covers auth, secrets, payments, migrations and deploys. Three reviewer prompts (product, safety, architecture) each read the diff, and any one of them can block it. By default it runs in shadow mode, which records what it would have merged and stops. When you turn merging on, the default limit is 1 merge a day. The fixer runs with the credentials it needs and nothing else. It's open source as [claude-bug-squash](https://github.com/jddavenportOpen/claude-bug-squash).

**One that failed.** An adversarial audit I ran found the review panel itself was broken. The three-of-three quorum passed at one-of-three. The "(3/3 approved)" line in the receipt was hardcoded. And the parser took the last word of the reply as the verdict. I fixed all three. Now the receipt is computed from the reviews that ran.

**What it costs.** It's slow on purpose. Shadow mode is on by default, and most fixes still land through me.

## 4. What the models cost

Measured at list prices over one stretch in July 2026, Opus was 93% of my model usage ($1,248 of $1,349, across 10,570 turns). Every change ran on the top model, review included. A typo fix got the same review as a change to what the agents are allowed to do on their own. I never measured what that review caught.

Now review effort depends on how much a change can break. Changes to autonomy, safety checks, credentials, outside sends or anything you can't undo get the top model and a second reviewer. Normal engineering gets normal review. Doc fixes get the builder's own checks and no second model. A lint rule flags workflows where every step runs on the top setting.

## The restart incident

On 2026-07-01 at 15:31, all 12 live agent sessions died within 2 seconds.

A session had just changed the chat bridge, the process every interactive session runs under. To load its change, it restarted the bridge with `launchctl kickstart -k`. That kills the whole process group. Every session died, including the one that ran the command. Its own shell call came back with exit code 137. Work in progress was lost.

A safe restart script that checks for live sessions had existed for weeks. Nothing made agents use it. The service file even called the raw command "the proven path", and an agent read that literally. The rule lived in a doc, and an agent restarted the service anyway.

What I shipped the same day:

- A hook that blocks the raw restart commands, with an override token for when I mean it.
- A detector that pages me when 4 or more sessions die inside 120 seconds, and names the session that ran the command.
- Then the written rule.

Two fixes I turned down, with reasons written down. Restarting the bridge to "apply" the fix would repeat the incident, and the hook doesn't need a restart. Letting sessions survive a bridge kill would leave orphans nothing can re-adopt.

What's still open: the hook only binds agent sessions. I can still kill everything from my own terminal. That's why the detector exists.

## What I'd tell someone deploying agents at work

1. Put each kind of action you can't undo behind one gate, and fail the build if code goes around it.
2. Tie an approval to the exact action, single-use and expiring.
3. If a rule matters, have a hook enforce it. Every incident here traces back to a rule that only existed as text.
4. When a gate blocks something it shouldn't, fix the caller or sharpen the gate. Don't widen it because you're behind.
5. Have every check read the diff itself.
6. Test your checkers. My review panel shipped with a broken quorum and a hardcoded receipt.
7. Spend review effort where a change can do damage. A typo fix doesn't need the top model.
8. Keep a person where it matters. [recruit-copilot](https://github.com/jddavenportOpen/recruit-copilot), the open version of my job-search tool, never submits an application.
9. Write the retro with receipts: the command, the transcript line, the exit code, and what the fix still doesn't cover.

## What's still open

- Texts don't have per-message approval tokens yet.
- The restart hook doesn't cover a human terminal.
- The three standing approvals send without a per-message yes, inside their caps.
- I never measured what top-tier review actually caught.

---

*I live in Provo, Utah. I founded and run the BYU AI Foundry, and I'm an MBA candidate at BYU Marriott. Before that I was a Principal PM at National Grid, where I grew an internal cloud services platform from $2M to $20M+ a year in chargeback. Most of the code behind this system was written by Claude Code agents I run. I pick what gets built, set the tests it has to pass, and check it by using it.*
