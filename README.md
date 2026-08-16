# Deploying a 30-Agent AI Organization Safely in Production

*A capability-vs-risk case study. JD Davenport, 2026.*

**Abstract.** This is an account of running a production AI organization, roughly 38 live agents across 22 domains at the time of writing, with real-world blast radius: agents that send email as me, fix code autonomously, and drive a corporate laptop remotely. For each capability it names the concrete risk, the control that bounds it, the tradeoff accepted, and one incident where the controls failed and what changed as a result.

## The system, honestly scoped

Since late 2025 I have operated a personal AI organization: about 38 live agents across 22 domains (health, finances, recruiting, school, consulting, a corporate work pilot), over 100 built and retired across the project's life. It runs on a Mac Mini against a 353-table Postgres backbone, 13 MCP servers, and roughly 117 scheduled loops, driven through a web cockpit and Telegram, with over ten thousand tests in the codebase.

Two caveats matter more than the numbers. This is a single-operator system with one principal; it does not solve multi-tenant authorization and I will not pretend it does. And "production" here means real consequences rather than enterprise SLAs: real email to real people, real spending surfaces, real corporate hardware. That blast radius is what makes the safety engineering non-optional, and why the lessons transfer even though the scale does not.

The organizing principle: every capability was built twice. Once as a demo, which is cheap. Once as a bounded system with an enforcement layer the agent cannot talk its way past, which is the work.

## Capability 1: Agents that act on a live machine

**Risk.** Dozens of concurrent sessions hold shell access on the box that also holds my credentials, my family's data, and every other agent's state: destructive commands, credential exfiltration, sibling interference.

**Controls.** Layered PreToolUse hooks that run before any command executes, deterministically, regardless of what the model believes. An ethos gate blocks top-tier secret reads, sends that bypass the approval rail, unattributed agent spawns, and force-pushes without an explicit allow flag (58 tests on the gate alone). A credential-scrape guard denies browser password stores, cookie databases, and keychain mass-dumps while allowing legitimate single-credential reads. And destructive paths fail closed. That last one was learned: a worktree-cleanup script had three content gates that all failed open, so an unreadable worktree read as empty and stale and was deleted; reproduction destroyed a never-committed file. Every gate in that path now fails closed.

**Tradeoff.** Hooks false-positive. The credential guard blocked its own documentation commit the day it shipped. The fix required access context (a real browser profile path or keychain command) rather than loosening the pattern. Gate friction is signal to sharpen the gate, never to remove it.

## Capability 2: Outbound communication as a real person

**Risk.** Agents send email and messages from my accounts to family, professors, and professional contacts. This is the most irreversible thing the system does; a wrong send cannot be rolled back.

**Controls.** Every external transport (email, iMessage, Slack DM) is wrapped by a single chokepoint function enforcing three things. A global freeze switch: sending defaults to frozen, with scoped, deliberate exceptions per rail. Content-bound approval: a send requires a single-use, expiring token bound to a SHA-256 hash of the exact recipients, subject, and body, so a "yes" approves one concrete message, a changed body invalidates the token, and a bare "yes" with no bound proposal is rejected. And an append-only audit ledger of every send and refusal.

The chokepoint is structural, not conventional: a CI guard fails the build if any code path transmits outside the gate. On first activation it caught five autonomous Gmail senders that manual audits had missed. Prose said everything used the gate; the machine check said otherwise.

The same principle now covers non-email actions. Early on, a two-day unsupervised "ship mode" run created 248 unwanted pull requests and 415 discussions on external GitHub repositories. Everything was closed and apologized for; the durable fix was a hook blocking fork, external-PR, and discussion creation without an explicit approval variable.

**Tradeoff.** Fail-closed friction, sometimes for weeks. One approved university email rail silently refused every send because the token-minting caller and the transport disagreed on the binding key. The fix threaded the correct key through the call chain. The binding was not loosened. A gate that relaxes under schedule pressure is not a gate.

## Capability 3: Autonomous code fixing

**Risk.** A pipeline that reproduces a bug, writes a fix, and can merge it without a human can merge anything: an auth bypass, a disabled safety rail, a sincere mistake. And an agent asked to certify its own work will certify it.

**Controls.** Nine independent layers, each re-checking the actual git diff rather than the agent's claims: triage with risk tiers; fixing in an isolated throwaway worktree; a mandatory red-to-green reproduction proof; blast-radius caps (at most 3 files and 40 changed lines, no new dependencies, no schema, no CI or infrastructure, no binaries); a never-touch deny-list (auth, secrets, payments, migrations, deploy, safety rails) whose match is a hard block; a three-seat adversarial review panel (Product, Safety, Architecture, each a scoped read-only model call, any one veto blocks); shadow mode that records what would have merged and stops, on by default; a per-day auto-merge limit, default 1; and an explicit arm switch. Out of the box it can never merge. The fixer subprocess also runs with an allowlisted environment: 178 credentials, including payment and infrastructure keys, are withheld from the one agent that could reach a merge. The model is open source as [claude-bug-squash](https://github.com/jddavenportOpen/claude-bug-squash) (MIT).

**The honest finding.** An adversarial audit found the review panel itself was broken: the three-of-three quorum passed at one-of-three, the "(3/3 approved)" receipt was hardcoded rather than computed, and the verdict parser took the last token instead of the verdict. All fixed, and the meta-lesson stuck: verifiers need verification, and receipts must be computed from evidence, not written by the agent being judged. Related work on calibrating LLM judges (anchored scales, strong-vs-mismatch separation, self-consistency) is published at [mcp-judge](https://github.com/jddavenportOpen/mcp-judge).

**Tradeoff.** Throughput is deliberately poor: one auto-merge a day, shadow by default, most fixes still landing through a human. That is the price of stating precisely what the system cannot do.

## Capability 4: Driving a corporate machine remotely

**Risk.** With my employer's sanction, the fleet drives a Claude Code instance on a locked-down corporate laptop through a relay, at about 12 seconds round trip versus a roughly 100-second baseline. This is someone else's hardware and intellectual property; failure modes that are annoying on my own machine are unacceptable here.

**Controls.** A PreToolUse deny-hook on the corporate side blocks destructive shell commands (the rm -rf, dd, mkfs class) even when the session runs with permission prompts disabled, because a hook fires before execution regardless of session mode. Corporate IP and personal work are separated by charter and storage boundary. Trust is deliberately one-directional: when the relay's pull leg degraded, the accepted fix was to let it stay degraded and rely on the push leg plus a dead-man alarm that pages if the mirror goes quiet for 36 hours, rather than open inbound network exposure on a corporate device.

**Tradeoff.** A hard capability ceiling and real latency. The fleet can do less on that machine than it technically could. That is the point.

## Cost routing as a governance surface

Model spend turned out to be governance wearing an economics costume. Measured on one day: $1,349 in list-price-equivalent usage across 10,570 turns, 93 percent on the largest model, because every pipeline lane defaulted to top-tier adversarial review regardless of what it touched. A documentation fix got the same maximum-effort review as an autonomy increase.

The repair was a lint-enforced rule: review effort scales with blast radius, not activity. Changes touching autonomy, safety gates, credentials, external sends, or anything irreversible get the top tier and an independent verifier. Normal engineering gets standard review. Mechanical and doc-only changes get the builder's own checks and no second model. Rote loops are pinned to small models at low effort. The accepted tradeoff is explicit: less scrutiny on low-stakes changes buys scrutiny that means something where it counts.

## What broke and what we changed: the restart incident

On 2026-07-01 at 15:31, all 12 live agent sessions died within two seconds.

A session had just committed a change to the chat bridge, the process hosting every interactive session as a PTY child. To load its change it restarted the bridge with a raw `launchctl kickstart -k`, which SIGKILLs the whole process group. Every session died, including the one that ran the command: its own shell tool returned exit code 137, killed by its own restart. In-flight work was lost.

The five-whys held up a mirror. A guarded restart script with a live-session check had existed for weeks; nothing enforced its use. The service definition even advertised the raw command as "the proven path," and an agent reading it took it literally, which is what agents do. The safe path was documented; the unsafe path was not blocked. Honor-system prose lost to machine reality, and it always will.

Fixes shipped the same day, in order of importance: a PreToolUse hook blocking the raw restart class, with an explicit override token for deliberate operator use; a detector paging within five minutes on four or more session deaths inside 120 seconds, with culprit attribution from session transcripts; and only then the written rule. Two tempting fixes were rejected with reasons recorded: restarting the bridge to "apply" the fix (that repeats the incident, and the hook needs no restart), and letting children survive a bridge kill (orphaned sessions no supervisor can re-adopt are worse than dead ones). Residual risk is documented too: the hook binds only agent sessions, so a human terminal can still mass-kill, which is why the detector exists.

This incident, more than any design document, set the system's constitution: a rule that matters is a rule the machine enforces.

## Principles for deploying agentic AI in an enterprise

1. **One chokepoint per irreversible action class, enforced structurally.** CI fails if any code path circumvents it. Convention does not survive contact with generated code.
2. **Approve content, not intent.** A human "yes" binds to a hash of the exact action, single-use, expiring. Approving a category is how surprises happen.
3. **Machine enforcement beats prose.** Every incident here traces to a rule that existed only as text. Pre-execution hooks are the cheapest safety technology in this stack.
4. **Fail closed, and treat gate friction as signal.** Fix the caller or sharpen the gate. Never widen it under schedule pressure.
5. **Measure blast radius on the artifact.** Caps and deny-lists check the actual git diff, not the agent's account of it.
6. **Verify the verifiers.** My review panel shipped with a broken quorum and a hardcoded receipt. Receipts must be computed; judges must be calibrated and tested for their ability to fail things.
7. **Scale scrutiny with blast radius, not volume.** Uniform maximum review is not rigor; it is noise that trains everyone to ignore the reviewer.
8. **Keep the human where they are load-bearing.** The job-application copilot fills forms and stops; I click submit. Email sends on my one tap. The human action is small, concrete, and irreplaceable by design.
9. **Write the retro with receipts, including residual risk.** The mass-kill audit names the command, the transcript line, the exit code, and what the fix still does not cover. That document is worth more than the fix.

None of this makes the system safe in an absolute sense. It makes the failure modes enumerable, bounded, and paid for consciously, which is the most an honest engineer can claim and the least an enterprise should demand.

---

*JD Davenport builds agentic AI systems and previously led a $20M+ digital services P&L as a Principal Product Manager at a Fortune 500 utility. The governance model described here is open source: [claude-bug-squash](https://github.com/jddavenportOpen/claude-bug-squash) (MIT) and [mcp-judge](https://github.com/jddavenportOpen/mcp-judge).*
