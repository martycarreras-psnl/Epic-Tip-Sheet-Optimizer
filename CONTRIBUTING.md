# Contributing

This project welcomes contributions and suggestions. Most contributions require you to
agree to a Contributor License Agreement (CLA) declaring that you have the right to,
and actually do, grant us the rights to use your contribution. For details, visit
https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need
to provide a CLA and decorate the PR appropriately (e.g., status check, comment). Simply
follow the instructions provided by the bot. You will only need to do this once across
all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/)
or contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

## Getting Started

1. **Fork the repository** and clone your fork locally.
2. **Make your changes** on a feature branch.
3. **Open a pull request** against `main`.

## What We're Looking For

- New or improved Copilot skills for tip sheet ingestion, eval generation, or eval execution
- Improvements to existing skill prompts (better accuracy, fewer hallucinations, clearer instructions)
- Documentation clarity and typo fixes
- New process diagrams or visual assets
- Example eval outputs and scorecards
- Bug reports on skill behavior

## Guidelines

### Skill Files

- Skills live in `Skills/<skill-name>/SKILL.md`.
- Keep skill content prescriptive, not conversational.
- Follow the zero-hallucination principle — skills should instruct the LLM to extract only what is explicitly visible in source documents.

### Commits

- Use conventional commit messages: `fix:`, `feat:`, `chore:`, `docs:`.
- One logical change per commit. Squash fixup commits before merging.

### Assets

- Place process diagrams and images in `Assets/`.
- Use descriptive filenames (e.g., `TipSheetEndtoEnd.png`, not `diagram1.png`).

## Pull Request Checklist

- [ ] No secrets or credentials in the diff
- [ ] Commit messages follow conventional format
- [ ] PR description explains the why, not just the what
- [ ] Skill files include clear "When to Use" and "Process" sections

## Reporting Issues

Open a GitHub issue with:
- What you expected to happen
- What actually happened
- Steps to reproduce
- The skill and content you were working with

For security vulnerabilities, see [SECURITY.md](SECURITY.md).
