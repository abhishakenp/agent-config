---
name: dictation-optimizer
description: Optimize dictation habits using session transcript analysis and the Always voice daemon. Use when the user says "dictation optimizer", "fix my dictation", "optimize my speech", "vocabulary update", "grammar fix", "filler words", "repeated phrases", "how do I speak better", "always corrections", or wants to improve their voice-to-text accuracy and reduce redundant speech. Covers session-efficiency integration, Always daemon vocabulary/corrections, filler word filtering, and grammar optimization. No UI — skill + scripts + Always daemon.
---

# Dictation Optimizer — improve how you speak to AI

## Mental model

Your dictation habits directly affect token spend and session efficiency. Long prompts with filler words and repeated phrases waste tokens and make the AI's job harder. This skill:

1. **Analyzes** session transcripts for dictation issues (via session-efficiency)
2. **Identifies** specific filler words, repeated phrases, and overly long prompts
3. **Fixes** vocabulary and grammar via the Always daemon (`always vocab`, `always corrections`)
4. **Verifies** the fixes work

## Data flow

```
session-efficiency report → dictation issues → Always daemon corrections → verified fix
```

## Always daemon

The Always daemon (`~/proj/always`, binary at `~/.cargo/bin/always`) is a 24/7 voice activation system with:
- **Vocabulary management**: `always vocab extract` (from project), `always vocab import` (from STT software)
- **Corrections**: `always corrections list/approve/reject/clear/capture` — (wrong → right) pairs applied to `~/.always/glossary.json`
- **Speaker enrollment**: `always voice` — "My Voice" speaker verification
- **Glossary**: `~/.always/glossary.json` — context-aware corrections

## Script

`scripts/optimize-dictation.ts` — reads session-efficiency report, extracts dictation issues, generates correction plan, runs Always commands.

```bash
bun run scripts/optimize-dictation.ts [--days 30] [--dry-run] [--corrections corrections.json] [--apply]
```

- Without `--apply`: plan only, nothing written to the glossary.
- `--corrections <file>`: reviewed `(wrong → right)` pairs, authored by reading the transcripts:
  `[{"term": "Iris", "mistranscriptions": ["Irish"], "frequency": 53}]`
- With `--apply`: merges the corrections into `~/.always/glossary.json` after a timestamped backup, then reads the file back and checks every pair. Exits 1 if `--corrections` is missing, the file is invalid, or the read-back fails.
- `always vocab extract` is not used: the Always binary still stubs it, and it exits 0 on the stub.

## What it fixes

### Filler words
Common fillers: "like", "just", "basically", "actually", "really", "very", "so", "well", "um", "uh", "hmm", "right", "okay", "yeah", "i mean", "you know"

**Fix via Always**: Add to glossary as words to filter. Always's filter pipeline already throws away Whisper artifacts; extend it to filter filler words.

### Repeated phrases
Phrases repeated 3+ times in one prompt: "i want to", "a lot of", "we need to", etc.

**Fix**: These indicate the user is thinking out loud. The skill flags them and suggests condensing.

### Overly long prompts
Prompts > 100 words, especially > 500 words.

**Fix**: Break into shorter, focused prompts. Use the AI's ability to read files instead of dictating file contents.

### Redundant qualifiers
"very very", "really really", "just just" — saying the same qualifier twice.

**Fix**: Remove duplicates. The qualifier's intensity doesn't change with repetition.

## Integration with session-efficiency

1. Run `session-efficiency` to get the dictation issues report
2. Run `dictation-optimizer` to:
   - Parse the issues
   - Generate a correction plan
   - Optionally apply via Always daemon
3. Re-run `session-efficiency` to verify improvement

## Optimization recommendations

1. **Think before speaking**: State the goal in one sentence before dictating details.
2. **Don't repeat context**: The AI already has the file tree, recent commits, and project structure.
3. **Delegate, don't dictate**: "Fix the bug in X" instead of "I want you to read X and find the function and then edit it."
4. **Separate thinking from instructions**: Put reasoning in a separate message or skip it entirely.
5. **Use shorter prompts**: If > 100 words, you're probably over-explaining. The AI can ask clarifying questions.
6. **Filter filler words at the source**: Use Always's glossary to filter "like", "just", "basically" before they reach the AI.
7. **Use corrections for common mistakes**: If you frequently mispronounce a technical term, add a correction rule.
