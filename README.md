# Production Readiness Skill for Claude

A Claude skill that turns "review my code" into a production readiness check. It gives Claude a 150-item checklist across 15 areas, from authorization and database migrations to backups and prompt injection, plus a fixed format for reporting what it finds.

The goal is a review you can act on, not generic advice:

- Each finding comes with a severity, a fix, and its evidence, including the file and line when the issue is in the code.
- Items that don't fit your project's stage yet are marked "not applicable yet", with the trigger that would make them relevant.
- Passing items aren't listed as findings, so the report stays short.

## What it covers

| # | Area | Items |
|---|---|---|
| 1 | Security and access control | 22 |
| 2 | Data and databases | 14 |
| 3 | Concurrency and distributed systems | 14 |
| 4 | Resilience and fault tolerance | 12 |
| 5 | Messaging, background work, and real time | 11 |
| 6 | Traffic, edge, and networking | 9 |
| 7 | Caching | 4 |
| 8 | Performance, scaling, and cost | 13 |
| 9 | APIs and versioning | 5 |
| 10 | Observability | 10 |
| 11 | Testing and code quality | 5 |
| 12 | Delivery, deployment, and infrastructure | 14 |
| 13 | Disaster recovery and incident response | 8 |
| 14 | Client side | 4 |
| 15 | AI and LLM features (when the app calls models) | 5 |

## How it works

**It matches your stage.** The skill applies what fits the project's stage. It uses the stage tiers in your `CLAUDE.md` if you define them. If not, it infers one of Prototype, MVP, Production, or Scale and tells you which one it assumed.

**When it kicks in.** Claude decides when to load a skill from its description (Anthropic, n.d.-a, n.d.-b). This one covers code reviews, security or reliability audits, production readiness checks, and the moment before you design or change authentication, data models, payments, external integrations, background jobs, caching, or deployment. If it doesn't load on its own in Claude Code, call it by name (see below).

**What you get back.** Reviews, audits, and readiness checks come back in six parts:

1. **Context:** the assumed stage, stack, and scope reviewed.
2. **Findings:** a table of Severity, Area, Finding, Evidence (file and line), and Fix, sorted by severity.
3. **Tier 0 status:** which non-negotiables were checked, and the result.
4. **Not applicable yet:** what was skipped for this stage, and what would make it relevant.
5. **Not verified:** anything that could not be inspected from the repository.
6. **Next steps:** the top 3 fixes, in priority order.

**Critical** means exploitable or losing data now. **High** means a likely outage, breach, or corruption. **Medium** means a reliability or maintainability risk. **Low** means worth doing later.

## Install

Claude Code reads personal skills from `~/.claude/skills/` and project skills from `.claude/skills/` (Anthropic, n.d.-b).

**Claude Code, for all your projects:**

```bash
git clone https://github.com/joechrisnaldy/production-readiness-skill.git ~/.claude/skills/production-readiness
```

**Claude Code, for one project** (run it from the project root):

```bash
git clone https://github.com/joechrisnaldy/production-readiness-skill.git .claude/skills/production-readiness
```

Or skip git and copy `SKILL.md` into a folder named `production-readiness` in either place. If you want to share the skill with your team through the project's own repository, copy the file rather than cloning, so it gets committed like any other file.

Claude Code picks up the skill in the current session, no restart needed. If the `skills` folder didn't exist when your session started, run `/reload-skills` once (Anthropic, n.d.-b).

To update later, run `git pull` in the folder you cloned into, for example `git -C ~/.claude/skills/production-readiness pull`. If you copied the file, copy the new version over it.

**Claude apps (claude.ai):** put `SKILL.md` in a folder named `production-readiness` and zip that folder. Then go to Customize > Skills, click **+**, choose **+ Create skill**, then **Upload a skill**, and upload the zip. Skills need code execution turned on, under Settings > Capabilities on individual plans (Anthropic, n.d.-c).

## Use it

Ask in plain words, for example:

- "Run a production readiness check on this repo."
- "Review the auth changes on this branch."
- "We're adding Stripe payments next week. What do we need before it ships?"

In Claude Code you can also call it by name: `/production-readiness` (Anthropic, n.d.-b).

## Optional: define your stage tiers

The skill refers to stage tiers in `CLAUDE.md` and reports a "Tier 0" line of non-negotiables, but it doesn't define either itself. If you use Claude Code and want the same rules applied on every review, add something like this to your project's `CLAUDE.md` (an example, adjust it to your project):

```markdown
## Stage tiers

Current stage: MVP

- Tier 0, non-negotiable at every stage: authorization on every endpoint, input validation,
  parameterized queries, secrets kept out of code, HTTPS, and tested backups.
- MVP: add rate limiting on auth endpoints, error tracking, and a rollback path for deploys.
- Production: add alerts, SLOs, structured logs, and a written recovery plan.
```

Without it, the skill still runs: it infers the stage and says so, and picks its own Tier 0 non-negotiables, which can vary between reviews. In claude.ai there is no `CLAUDE.md`, so tell Claude your stage in the chat.

## Credits

Adapted from content by [@nataindata](https://www.instagram.com/nataindata/) on Instagram. I turned it into this skill with Claude.

## License

MIT. See [LICENSE](LICENSE).

## References

Anthropic. (n.d.-a). *Agent Skills*. Retrieved November 8, 2026, from https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview

Anthropic. (n.d.-b). *Extend Claude with skills*. Retrieved November 8, 2026, from https://code.claude.com/docs/en/skills

Anthropic. (n.d.-c). *Use skills in Claude*. Claude Help Center. Retrieved November 8, 2026, from https://support.claude.com/en/articles/12512180-using-skills-in-claude
