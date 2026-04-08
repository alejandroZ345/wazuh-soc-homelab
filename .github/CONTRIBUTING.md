# Contributing

Thank you for your interest in contributing to this project. Whether you're fixing a typo, improving a detection rule, or suggesting an entirely new phase — contributions are welcome.

---

## How This Project Works

This is a personal SOC home lab documented as a portfolio project. Each phase builds on the previous one, and all documentation follows a consistent structure and style. Contributions should align with that structure to maintain coherence across the repository.

Before contributing, please review the [Code of Conduct](.github/CODE_OF_CONDUCT.md).

---

## What You Can Contribute

**Documentation improvements** — Typos, grammatical corrections, broken links, formatting inconsistencies, or unclear explanations in any phase file, runbook, or detection mapping.

**Detection rule improvements** — Optimizations to existing XML rules (100001–100005), suggestions for reducing false positives, or alternative detection logic for the same threat vectors.

**Runbook enhancements** — Additional triage steps, improved containment procedures, or expanded false positive guidance based on real-world SOC experience.

**MITRE ATT&CK mapping corrections** — If a technique mapping is inaccurate or a more precise sub-technique exists for a detection, corrections to `detections/mitre-attack-map.md` are especially valuable.

**New phase suggestions** — Ideas for future lab phases (active response, SOAR integration, log forwarding, etc.) can be submitted as issues using the Phase Suggestion template.

---

## Style & Conventions

To keep the repository consistent, please follow these conventions in any pull request:

**File naming** — Phase files use `phase-N-short-description.md`. Runbooks use `runbook-RULEID-short-description.md`. All filenames are lowercase with hyphens.

**Markdown structure** — Each phase file follows a standard layout: blockquote objective at the top, numbered sections with `##` headers, code blocks with language hints, and navigation links at the bottom (`Previous / Next`).

**Tone** — Technical and precise, written in third person or imperative mood. Avoid marketing language. Document what was done and why, including failed attempts and troubleshooting — the process matters as much as the result.

**Detection rules** — Any XML rule contribution should include the rule logic, an explanation of why it works, and validation evidence (what triggers it, what doesn't).

**Runbooks** — Follow the established structure: header block (trigger, description, ATT&CK mapping), Context section, numbered Triage Steps with embedded commands, and a False Positive Guidance section at the end.

---

## Submitting Changes

1. **Fork** the repository and create a branch from `main` with a descriptive name (e.g., `fix/phase-5-typo` or `feature/runbook-100006`).
2. **Make your changes** following the conventions above.
3. **Test your changes** — If modifying detection rules, include validation evidence. If modifying documentation, verify all internal links work.
4. **Open a Pull Request** against `main` with a clear description of what changed and why.

For small fixes (typos, broken links), a single commit with a clear message is sufficient. For larger changes (new runbooks, rule modifications), please reference the relevant phase or issue in the PR description.

---

## Reporting Issues

Use the issue templates provided in this repository:

- **Bug / Inconsistency Report** — For documentation errors, broken links, inaccurate MITRE mappings, or any content that doesn't match the actual lab behavior.
- **Phase Suggestion** — For proposing new lab phases, detection coverage expansions, or structural improvements to the repository.

If your report doesn't fit either template, a blank issue is fine — just provide enough context for the maintainer to understand and act on it.

---

## Questions?

If you're unsure whether a contribution is appropriate or need clarification on the project's direction, open an issue and ask. There are no bad questions.
