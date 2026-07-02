# Review

## Findings

1. **High - The Stop hook runs an all-changes Codex review after every Claude stop.**  
   `.claude/independent-reviewer/plugins/independent-reviewer/hooks/hooks.json:3-9` installs an unconditional `Stop` hook that runs `codex exec "Please review all changes since the last commit and write feedback to planning/REVIEW.md"`. When this plugin is enabled, every Claude response can spawn a Codex review, spend time/tokens, and overwrite `planning/REVIEW.md` even when the user did not ask for a review. This run was itself launched from a `Stop` hook, so the behavior is already observable. Make review execution an explicit command/agent action, or add a narrow guard so the hook only runs for an intentional review workflow.

2. **High - `codex-reviewer` is missing valid YAML frontmatter.**  
   `.claude/agents/codex-reviewer.md:1-4` starts with a blank line and then `name:`/`description:` without an opening `---`; it only has the closing delimiter on line 4. Claude agent definitions need the frontmatter block to start at the top of the file, as the other agent files do. As written, this agent will likely be ignored or treated as plain prompt text. Add `---` at line 1 and remove the leading blank line.

3. **Medium - The `review-file` slash command is not a usable command prompt.**  
   `.claude/commands/review-file.md:1-8` contains JSON-like plugin metadata with a trailing comma, not instructions for reviewing a file. If invoked as `/review-file`, Claude will receive that manifest text instead of a review task; if treated as JSON, it is invalid because of the trailing comma. Move plugin metadata under the appropriate `.claude-plugin/plugin.json`, or replace this file with the intended command prompt.

4. **Medium - Multiple review automations write the same file and will duplicate each other.**  
   `.claude/agents/code-reviewerall.md:8-16`, `.claude/independent-reviewer/plugins/independent-reviewer/agents/independent-reviewer.md:8-16`, and the Stop hook all run the same `codex exec` command targeting `planning/REVIEW.md`. If a user invokes the agent while the hook is enabled, one review can run from the agent and another can run immediately afterward from the Stop hook, with the last writer winning. Keep one canonical trigger for the all-changes review, or give separate workflows separate output files and clear names.

5. **Medium - Review output paths and generated artifacts are inconsistent.**  
   `.claude/agents/codex-reviewer.md:3` and `.claude/agents/codex-reviewer.md:8` write to `planning/REVIEWX.md`, while `.claude/agents/code-reviewer.md:9`, `.claude/agents/code-reviewerall.md:12`, and the plugin hook write to `planning/REVIEW.md`. The working tree also contains untracked generated review files at `planning/REVIEWX.md` and `planning/REVIEW_old.md`. Pick one canonical output contract, or document which review mode owns each file, and avoid committing stale scratch outputs by accident.

6. **Low - `codex-reviewer` contains visible mojibake.**  
   `.claude/agents/codex-reviewer.md:7` contains `â€“` where an en dash or plain hyphen was intended. Replace it with ASCII punctuation or re-save the file as valid UTF-8 text.

7. **Low - `.claude/settings.json` has a noise-only formatting diff.**  
   `.claude/settings.json:6-9` only adds blank whitespace and removes the final newline. The JSON still parses, but this creates an unnecessary diff. Restore the final newline and remove the extra blank lines before committing.

## Notes

- The README update is consistent with the current repo state: it no longer claims the full Docker/frontend stack already exists and points readers to the completed market-data subsystem.
- The new market planning documents appear consistent with the current `backend/app/market/` implementation at review depth; I did not find a blocking issue there.
