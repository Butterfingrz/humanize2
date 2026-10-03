# AGENTS.md

For implementation:

- MUST pass all tests and linters before merging a PR or claiming the task is done.
- MUST ensure CI is green before merging a PR or claiming the task is done.
- PREFER use popular and well-maintained libraries rather than custom implementations.

For `specs/*.md`:

- MUST strictly adhere to specs at `specs/`.
- MUST NOT modify any SPEC UNLESS explicitly instructed to do so.
- MUST ensure code is minimal and any further simplification will lead to dissatisfaction of the spec.

For version control:

- MUST adhere to [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/).
- MUST delete local branches or worktrees and remote branches once merged.

For docs:

- MUST update docs once any impl changes to avoid misalignment between code and docs.
- MUST adhere to the minimal spec of [Standard Readme](https://raw.githubusercontent.com/RichardLitt/standard-readme/refs/heads/main/spec.md) for `README.md`.
