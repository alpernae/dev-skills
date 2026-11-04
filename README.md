# dev-skills

A collection of [Agent Skills](https://agentskills.io) for AI coding agents — Claude Code, Cursor, Codex, OpenCode, and any other tool that reads a `SKILL.md` file.

Each skill is a self-contained folder with a `SKILL.md` (and optional `references/` or `rules/`) that an agent loads when a task matches its description.

## Skills

### Original skills

These skills are authored and maintained in this repository.

| Skill | Description |
| --- | --- |
| [`supabase-auth-mobile`](./supabase-auth-mobile) | Supabase Auth for Expo and React Native apps: email sign-up and confirmation, password sign-in, TOTP MFA with recovery codes, password reset, session gating, account management, and account deletion. |
| [`supabase-auth-web`](./supabase-auth-web) | Supabase Auth for React web apps (Vite/CRA and Next.js with `@supabase/ssr`): the same account flows with cookie-based sessions, middleware session refresh, and `/auth/confirm` link handling. |

### Third-party skills

These skills are included from public upstream projects and were not authored by this repository's maintainer. Each entry links to the original source. Please use the upstream repository as the source of truth for updates and licensing.

| Skill | Source | Author | License |
| --- | --- | --- | --- |
| [`brandkit`](./brandkit) | [Leonxlnx/imagegen-skills — `skills/brandkit`](https://github.com/Leonxlnx/imagegen-skills/tree/main/skills/brandkit) (originally in [`Leonxlnx/taste-skill`](https://github.com/Leonxlnx/taste-skill)) | [Leonxlnx](https://github.com/Leonxlnx) | MIT |
| [`expo-animation`](./expo-animation) | [expo/skills — `plugins/expo/skills/expo-animation`](https://github.com/expo/skills/tree/main/plugins/expo/skills/expo-animation) | [Expo](https://github.com/expo), in collaboration with [Emil Kowalski](https://github.com/emilkowalski) | MIT |
| [`expo-design-system`](./expo-design-system) | [expo/skills — `plugins/expo/skills/expo-design-system`](https://github.com/expo/skills/tree/main/plugins/expo/skills/expo-design-system) | [Expo](https://github.com/expo) | MIT |
| [`expo-router`](./expo-router) | [expo/skills — `plugins/expo/skills/expo-router`](https://github.com/expo/skills/tree/main/plugins/expo/skills/expo-router) | [Expo](https://github.com/expo) | MIT |
| [`frontend-design`](./frontend-design) | [anthropics/skills — `skills/frontend-design`](https://github.com/anthropics/skills/tree/main/skills/frontend-design) | [Anthropic](https://github.com/anthropics) | Apache-2.0 (see [`LICENSE.txt`](./frontend-design/LICENSE.txt)) |
| [`react-native-skills`](./react-native-skills) | [vercel-labs/agent-skills — `skills/react-native-skills`](https://github.com/vercel-labs/agent-skills/tree/main/skills/react-native-skills) | [Vercel](https://github.com/vercel) | MIT |
| [`remotion-best-practices`](./remotion-best-practices) | [remotion-dev/skills — `skills/remotion-best-practices`](https://github.com/remotion-dev/skills/tree/main/skills/remotion-best-practices) | [Remotion](https://github.com/remotion-dev) | No license file in the upstream skills repository — see [Remotion licensing](https://www.remotion.dev/license) |
| [`taste-skill-v1`](./taste-skill-v1) | [Leonxlnx/taste-skill-classics — `skills/taste-skill-v1`](https://github.com/Leonxlnx/taste-skill-classics/tree/main/skills/taste-skill-v1) | [Leonxlnx](https://github.com/Leonxlnx) | MIT |

> **Note:** The copies here are snapshots. They may lag behind the upstream repositories, and any local edits are this repository's responsibility, not the original authors'. Check the source links for the latest versions and license terms.

## Using a skill

Copy a skill folder into the skills directory of your agent (for example, `.claude/skills/` for Claude Code or `~/.config/opencode/skills/` for OpenCode), or use the [Skills CLI](https://skills.sh):

```bash
npx skills add alpernae/dev-skills --skill supabase-auth-mobile
```

Then ask your agent for a task the skill covers. Skills load automatically when their description matches the request.

## License

The original skills (`supabase-auth-mobile` and `supabase-auth-web`) are released under the MIT license, as declared in their `SKILL.md` files. Third-party skills remain under their respective upstream licenses listed above.
