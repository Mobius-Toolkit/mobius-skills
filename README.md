# mobius-skills

This repository holds the skills of the agents of Mobius. Mobius copies the skills of `main` into the skill folder of each agent session.

## Layout

- A general skill is a top-level folder with a `SKILL.md` file, for example `mobius-local-state/`. Each session gets it.
- A repository skill is in `repos/<repository name>/<skill name>/SKILL.md`. The repository name is the name of the repository in the organization, for example `repos/Mobius/task-overlap-check/` for Mobius-Toolkit/Mobius. Only a session of that repository gets it, in the same skill folder as the general skills.
- When a repository skill and a general skill have the same name, the session gets only the repository skill.
- A session with no repository (the Coordinator) gets only the general skills.
- A Lead session gets the repository skills of the repository of its Workstream issue.
- A skill can name the roles that get it, with the key `roles` in the `metadata` map of the frontmatter of `SKILL.md`. The value is a comma-separated list of the role names of the Mobius config: `lead`, `coordinator`, `triager`, `implementer`, `researcher`, `reviewer`, `judge`, `curator`, `diagnoser`. A skill with no `roles` key goes to each role.

  ```
  ---
  name: task-overlap-check
  description: …
  metadata:
    roles: lead
  ---
  ```

- When a repository skill does not go to the role of a session, that session also does not get the general skill of the same name.
- A `SKILL.md` file in a different place is not a skill.
