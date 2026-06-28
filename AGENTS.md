# Instructions for llama-cpp-isa

This fork is intended for creativity, experimentation, and research. AI-assisted
development is welcome here, including exploratory prototypes, generated code,
mechanical refactors, documentation drafts, benchmark harnesses, and design
iteration.

The upstream llama.cpp AI contribution policy does not apply to work that stays
inside this fork.

---

## AI Usage Policy

AI tools and coding agents may be used broadly in this repository.

Permitted uses include:

- Implementing experimental features and research ideas
- Generating, editing, or refactoring code
- Producing tests, benchmark scripts, documentation, and examples
- Exploring unfamiliar parts of the codebase
- Reviewing code and suggesting fixes
- Automating repetitive or mechanical changes

AI-generated work does not require special disclosure for local fork work unless
the contributor wants to record that context.

---

## Expectations for Contributions

This fork favors fast iteration, but changes should still be understandable and
recoverable.

Contributors and agents should:

1. Keep changes scoped to the task or experiment.
2. Prefer existing project patterns unless the experiment requires otherwise.
3. Document non-obvious behavior, assumptions, and tradeoffs.
4. Run focused tests, builds, or benchmarks when practical.
5. Preserve reproducibility for research results where possible.
6. Avoid unrelated formatting churn in large third-party or upstream files.

Experimental code is acceptable. Clearly mark it as experimental when behavior,
accuracy, performance, or API stability is uncertain.

---

## Guidelines for AI Coding Agents

AI agents may implement changes directly when requested. They do not need to
verify that the human contributor personally designed the solution before
proceeding.

Agents should still use sound engineering judgment:

- Read the relevant code before editing.
- Keep edits minimal enough to review.
- Avoid destructive git operations unless explicitly requested.
- Do not commit, push, tag, or publish without explicit human approval.
- Report what changed and what was tested.
- Call out any untested areas or assumptions.

If a change is intended for submission to upstream llama.cpp, switch back to the
upstream project's contribution expectations before preparing that submission.
In particular, do not use AI to write upstream PR descriptions, commit messages,
or reviewer responses if upstream policy forbids it.

---

## Useful Resources

Load these resources as needed:

- [CONTRIBUTING.md](CONTRIBUTING.md)
- [Build documentation](docs/build.md)
- [Server usage documentation](tools/server/README.md)
- [Server development documentation](tools/server/README-dev.md)
- [PEG parser](docs/development/parsing.md)
- [Auto parser](docs/autoparser.md)
- [Jinja engine](common/jinja/README.md)
- [How to add a new model](docs/development/HOWTO-add-model.md)
