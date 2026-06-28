# Privacy Manifest

This file is the source of truth for the content pipeline's privacy review.
Every generated post must pass a review against these rules before staging.

## PUBLISH freely

These topics can be discussed openly with technical detail:

- CLAUDE.md structure: what sections to include, how to organize instructions
- Memory system architecture: types (user, feedback, project, reference), index management, when to save vs skip, staleness handling
- Hook system design: pre/post tool use hooks, write guards, canary watchdog pattern, degradation detection
- Skill design patterns: how skills chain together, session-start/wrapup lifecycle, skill triggers
- Task management patterns: sizing conventions (#quick/#medium/#deep), project note structure, progress tracking
- Automation architecture: heartbeat files, launchd scheduling, health monitoring, self-healing loops
- Review cadences: weekly/monthly/quarterly review workflows, what each covers
- File organization philosophy: folders over tags, index notes, three-tier storage (active/reference/archive)
- MCP tool integration: how to wire external services, permission architecture
- Deployment patterns: Docker on NAS, tar pipe deployment, volume management
- Content pipeline design: autonomous content generation, privacy review, staging gates
- Orchestration layer design: check registries, scheduled runners, auto-heal actions
- Low-friction system design: reducing activation energy, matching tasks to energy levels, ADHD-friendly patterns (frame as universal design, not diagnosis)
- Security patterns: permission tiers, write guards, file operation safety, NAS safety rules
- Prompt engineering for system prompts: reliability canaries, instruction layering, context management
- Project lifecycle: scaffolding, progress tracking, close-out, archiving patterns
- Wikilink conventions: filename matching, display aliases
- Template systems: standardized note formats, frontmatter patterns
- Vault structure: pillar-based organization, inbox capture, journal hierarchy
- Config management: YAML configs, environment variables, settings layering

## NEVER publish

These categories must be completely stripped. If a post touches any of these, remove the specific details and replace with generic examples:

- **Names:** Any real person's name (the system builder, family members, friends, colleagues, pets). Use "the builder" or "the user" instead.
- **Location:** City, neighborhood, street, address, region. Use "their area" or omit.
- **Financial:** Salary, debt amounts, account numbers, settlement details, specific dollar figures from personal finances. Architecture is fine, numbers are not.
- **Medical/Health:** Diagnoses (including neurodivergence framed as personal), medications, supplements, specific health conditions, provider names. Teaching executive-function-friendly design is fine. Saying the builder has a specific condition is not.
- **Employment:** Company name, role title, team name, compensation, colleagues. "A payments role" is fine. The specific company is not.
- **Family:** Relationship structure, custody details, family dynamics, estrangement details.
- **Legal:** Estate plans, trust contents, insurance specifics, legal disputes, case details.
- **Credentials:** API keys, tokens, passwords, IP addresses, port numbers tied to personal infrastructure. Generic examples (port 3000) are fine. Actual infrastructure IPs are not.
- **School:** Institution name, enrollment dates, degree specifics. "Pursuing a graduate degree" is fine.
- **Vault paths that reveal personal content:** `/2_Pillars/Financial/` as a structural example is fine. A path that contains a person's name or specific financial document is not.
- **Personal preferences:** Music taste, dietary details, hobbies, pet details. These don't belong in a systems blog.

## TRANSFORM (teach the concept, strip the personal)

These topics have publishable architectural value but contain personal details that must be generalized:

| Raw concept | Publish as |
|---|---|
| ADHD/AUDHD-specific design | "Executive-function-friendly design" or "low-friction design for busy professionals" |
| Financial dashboard with Monarch Money | "Financial dashboard architecture with API integration" (no provider name, no amounts) |
| People/relationship management system | "Contact relationship management pattern" (no actual people) |
| Health tracking with Oura/Eight Sleep | "Wearable data integration pattern" (no specific devices unless reviewing them generically) |
| Employer-specific workflows | "Workplace onboarding automation" (generalized) |
| Pet care tracking | "IoT/recurring-task tracking pattern" (generalized) |
| Personal schedule automation | "Calendar and task automation" (no specific events) |
| Estate planning system | "Document management for legal compliance" (no details) |

## Review checklist

Before a post is staged, verify:

1. No real names appear anywhere in the post
2. No location more specific than "United States" appears
3. No financial figures from personal accounts appear
4. No medical conditions are attributed to the builder
5. No employer is named
6. No family structure details appear
7. No API keys, tokens, or real IP addresses appear
8. Architectural patterns are described generically enough to be useful to anyone
9. The post would be safe to share with a stranger, a colleague, or a future employer
10. The builder could not be uniquely identified from the post content alone
