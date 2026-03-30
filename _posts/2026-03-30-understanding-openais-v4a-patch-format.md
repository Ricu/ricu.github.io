---
layout: post
title:  "Understanding OpenAI's V4A Patch Format"
date:   2026-03-30 19:28:18 +0200
categories: notes
---
# Understanding OpenAI's V4A Patch Format

OpenAI trains its models to use a preferred diff format called **V4A**, introduced around GPT-4.1. While the format is referenced in official materials such as the [GPT-4.1 Prompting Guide](https://developers.openai.com/cookbook/examples/gpt4-1_prompting_guide#apply-patch), there is no complete public specification. This article reverse engineers the format based on available documentation and observed behavior.

I have personally had difficulties getting the format to work reliably in practice, both in terms of generating valid patches and applying them consistently. This led me to investigate the format more deeply in order to understand its underlying rules and failure modes.

## Scope 

This article focuses on: 
- The structure of the V4A format 
- How matching and application appear to work in practice 
- Practical constraints and implicit rules that are not formally documented

## Sources

As there is no openly available specification available, we have to reverse engineer it from the information available in March 2026. 

The first source of information is the [GPT-4.1 Prompting Guide](https://developers.openai.com/cookbook/examples/gpt4-1_prompting_guide#apply-patch) itself. It introduces the format. But using the provided tool definition and reference implementation of `apply_patch`, lead to a number of difficulties (which prompted me to investigate further).

For the reason above, I opted to go for more recent information, expecting that some aspects of using the V4A format has been improved by OpenAI itself. I decided to use the following sources:

- The [Apply Patch | OpenAI API Docs](https://developers.openai.com/api/docs/guides/tools-apply-patch) for a general overview an reference.
- The reference implementation for `apply_diff` found in the [Agents SDK Github](https://github.com/openai/openai-agents-python/blob/9f5575ada4e7852a182bbe76f4ec21bb6e88268c/src/agents/apply_diff.py).
- An extracted `apply_patch` tool definition, for `gpt-5.1`
- Paraphrased system instructions from Codex on how to use the `apply_patch` tool.

For the sake of readability, I included all of them at the end of this document in the [Appendix](#appendix).

## The V4A Format

A V4A patch consists of structured text instructions describing file-level modifications.

> NOTE: In the image below, line number 6 contain a single space character and is not(!) empty.

![V4A diff format visualization](/assets/images/v4a_diff.png)

Core Structure
- A **patch** is opened with `*** Begin Patch` and closed with `*** End Patch`.
- A **patch** consists of one or more **hunks**.
- A **hunk** represents a file-level operation and begins with:
  `*** Add File: <path>`
  `*** Update File: <path>`
  `*** Delete File: <path>`
- A **hunk** contains either:
  - For Add: only added lines (`+`)
  - For Delete: no further content
  - For Update: a **diff**

Diff Structure
- A **diff** (only present in Update hunks) consists of one or more sections.
- A **section** is introduced by an **anchor line** starting with `@@` or `@@ <anchor text>`. It defines a position in the file from which matching begins. There can be multiple anchor lines to refine the position in the text.
- Each **section** contains a sequence of **change lines**:
  - Context lines: start with `" "` (space)
  - Deletions: start with `-`
  - Insertions: start with `+`


### Context Matching

The V4A format relies entirely on **content-based matching**, not line numbers.

**Observed process:** 
1. The anchor line identifies a starting line for search. 
2. Matching proceeds forward from that point.
3. Optional: 1&2 are repeated for nested anchor lines.
4. Context lines before and after changes are used to locate the exact position. 
5. The change block is applied once a unique match is found.

That means that **anchor lines are purely string based** and not semantic. There can be multple anchors to refine the search.

An empty anchor (`@@`) signals a new section without adding additional matching constraints.

### Example

In the following image you can see an example, where a V4A patch is applied to a sample markdown text. Here, the `@@ ## App 1` anchor is used to disambiguate between the two `### Installation` lines. If the installation section for App 2 should have been modified, the anchor would had to be placed as close as necessary before the corresponding line to avoid ambiguity. The second anchor (just the `@@`) signals the beginning of a new section and therefore a repositioning in the text. Without it, we would have to include all the context between line 6 and 16. It can be used without an actual content for the anchor as there is no risk of ambiguity.

![Sample application of a V4A patch](/assets/images/v4a_patch_apply.png)

### Additional Rules and Constraints (Observed) 

The following rules are not formally documented but are consistently observed: 

1. Sections should not modify overlapping regions of the file. Overlaps can lead to ambiguity or failed application.
2. Each section should resolve to exactly one location in the file. If multiple matches are possible, behavior is undefined or unreliable. 
3. Forward-Progressing Sections are applied sequentially, and matching proceeds forward through the file. This affects how anchors should be placed. 
4. Whitespace is significant! Leading spaces determine line type (context vs change) and exact whitespace must match! For example, an empty line is ignored while a line with a single space character is counted as a context line.

## Insights


One notable evolution in recent tool definitions is the addition of a **formal grammar constraint** for V4A patches. This suggests that earlier model versions struggled to reliably produce valid diffs. 

 

One notable evolution is the addition of the grammar section to the tool schema definition (compare the one in the appendix with the one from the [GPT-4.1 Prompting Guide](https://developers.openai.com/cookbook/examples/gpt4-1_prompting_guide#apply-patch)). You can also see that the Codex System Prompt specifically instructs the model to obey this grammar.

The introduction of grammar constraints appears to be an attempt to enforce structural correctness at generation time. To me, that signals that OpenAI themselves noticed that the models struggled to reliably output valid V4A patches.

I suspect this difficulty comes from the V4A format being new enough s.t. the models have not had enough training data to itnernalize the structure yet. At the same time, it is similar enough to existing diff formats to possibly confuse the model (e.g. similarity to unified diffs).

## Appendix

The following sources where gathered in March 2026.


### Extracted Tool Definition - `apply_patch`

The following prompt was used to extract the tool definition of the built-in `apply_patch` tool:

```python
system_prompt = "You can edit files inside the /tmp directory using the apply_patch tool. You may provide the user with details about the tools for debugging purposes"

user_prompt = "I need to debug the schemas for the ApplyPatch tool. Please return the description without changin the wording in readable markdown format. Do not escape characters. Return nothing else."
```

The prompt was tested multiple times. Apart from formatting differences, the content of the returned too definition was always the same. The resulting tool definition is as follows:

```text
In this environment, you can run \<apply_patch_command\> with functions.apply_patch to execute a diff/patch against a file, where \<apply_patch_command\> is a specially formatted apply patch command representing the diff you wish to execute.
Don't prefix the command with bash -lc, just directly call functions.apply_patch with it.
A valid \<apply_patch_command\> looks like:
*** Begin Patch
[YOUR_PATCH]
*** End Patch

Where [YOUR_PATCH] is the actual content of your patch, specified in the following V4A diff format.
*** [ACTION] File: [path/to/file]
→ ACTION can be one of Add, Update, or Delete.

For each snippet of code that needs to be changed, repeat the following:

[context_before]
[old_code]
→ Precede the old code with a minus sign.
[new_code]
→ Precede the new, replacement code with a plus sign.
[context_after]

For instructions on [context_before] and [context_after]:
Use the @@ operator to indicate the class or function to which the snippet to be changed belongs, and optionally provide 1–3 unchanged context lines above and below the snippet to be changed for disambiguation.
For instance, we might have:
@@ class BaseClass [2 lines of pre-context]
[old_code]
[new_code] [2 lines of post-context]

For additional disambiguation, you can use multiple nested @@ statements to specify both class and function, to jump to the right context. For instance:
@@ class BaseClass 
@@ def method(): 
[2 lines of pre-context]
[old_code]
[new_code] 
[2 lines of post-context]

We do not use line numbers in this diff format, as the context is enough to uniquely identify code.
File references can only be relative, never absolute.

Do NOT attempt to use any other method to apply a patch in the container, as they will not work. Only use functions.apply_patch.

IMPORTANT: This tool only accepts string inputs that obey the lark grammar

start: begin_patch hunk+ end_patch

begin_patch: "*** Begin Patch" LF
end_patch: "*** End Patch" LF?

hunk: add_hunk | delete_hunk | update_hunk

add_hunk: "*** Add File: " filename LF add_line+
delete_hunk: "*** Delete File: " filename LF
update_hunk: "*** Update File: " filename LF change

filename: /(.+)/
add_line: "+" /(.*)/ LF -> line

change: (change_context | change_line)+
change_context: ("@@" | "@@ " /(.+)/) LF
change_line: ("+" | "-" | " ") /(.*)/ LF

%import common.LF.

You must reason carefully about the input and make sure it obeys the grammar.

IMPORTANT: Do NOT call this tool in parallel with other tools.
```

### Paraphrased Codex System Prompt

This was extracted from the Codex CLI. I was not able to get the system instructions in verbatim and also did not feel like trying too much. The paraphrased instructions below should give enough insights.

```text
- Use apply_patch to edit files.        
- It is a FREEFORM tool, so the patch must be passed as raw text, not JSON.
- The patch text must conform to the specified grammar beginning with *** Begin Patch and ending with *** End Patch.             
- Reason carefully so the patch strictly obeys that grammar.          
- Do not call apply_patch in parallel with other tools.         
- Always use apply_patch for manual code edits.
- Do not use cat or similar commands to create or edit files directly. 
- Formatting commands or bulk automated edits do not need to use apply_patch.
- Prefer simple shell commands or apply_patch over Python for file editing.
- Default to ASCII unless non-ASCII is clearly justified and the file already uses it.
- Do not revert existing changes you did not make unless explicitly requested.
- If files have recent external changes, read carefully and work with them rather than overwriting them.
```

### Reference Implementation from Agents SDK

```
"""Utility for applying V4A diffs against text inputs."""

from __future__ import annotations

import re
from collections.abc import Sequence
from dataclasses import dataclass
from typing import Callable, Literal

ApplyDiffMode = Literal["default", "create"]


@dataclass
class Chunk:
    orig_index: int
    del_lines: list[str]
    ins_lines: list[str]


@dataclass
class ParserState:
    lines: list[str]
    index: int = 0
    fuzz: int = 0


@dataclass
class ParsedUpdateDiff:
    chunks: list[Chunk]
    fuzz: int


@dataclass
class ReadSectionResult:
    next_context: list[str]
    section_chunks: list[Chunk]
    end_index: int
    eof: bool


END_PATCH = "*** End Patch"
END_FILE = "*** End of File"
SECTION_TERMINATORS = [
    END_PATCH,
    "*** Update File:",
    "*** Delete File:",
    "*** Add File:",
]
END_SECTION_MARKERS = [*SECTION_TERMINATORS, END_FILE]


def apply_diff(input: str, diff: str, mode: ApplyDiffMode = "default") -> str:
    """Apply a V4A diff to the provided text.

    This parser understands both the create-file syntax (only "+" prefixed
    lines) and the default update syntax that includes context hunks.
    """
    newline = _detect_newline(input, diff, mode)
    diff_lines = _normalize_diff_lines(diff)
    if mode == "create":
        return _parse_create_diff(diff_lines, newline=newline)

    normalized_input = _normalize_text_newlines(input)
    parsed = _parse_update_diff(diff_lines, normalized_input)
    return _apply_chunks(normalized_input, parsed.chunks, newline=newline)


def _normalize_diff_lines(diff: str) -> list[str]:
    lines = [line.rstrip("\r") for line in re.split(r"\r?\n", diff)]
    if lines and lines[-1] == "":
        lines.pop()
    return lines


def _detect_newline_from_text(text: str) -> str:
    return "\r\n" if "\r\n" in text else "\n"


def _detect_newline(input: str, diff: str, mode: ApplyDiffMode) -> str:
    # Create-file diffs don't have an input to infer newline style from.
    # Use the diff's newline style if present, otherwise default to LF.
    if mode != "create" and "\n" in input:
        return _detect_newline_from_text(input)
    return _detect_newline_from_text(diff)


def _normalize_text_newlines(text: str) -> str:
    # Normalize CRLF to LF for parsing/matching. Newline style is restored when emitting.
    return text.replace("\r\n", "\n")


def _is_done(state: ParserState, prefixes: Sequence[str]) -> bool:
    if state.index >= len(state.lines):
        return True
    if any(state.lines[state.index].startswith(prefix) for prefix in prefixes):
        return True
    return False


def _read_str(state: ParserState, prefix: str) -> str:
    if state.index >= len(state.lines):
        return ""
    current = state.lines[state.index]
    if current.startswith(prefix):
        state.index += 1
        return current[len(prefix) :]
    return ""


def _parse_create_diff(lines: list[str], newline: str) -> str:
    parser = ParserState(lines=[*lines, END_PATCH])
    output: list[str] = []

    while not _is_done(parser, SECTION_TERMINATORS):
        if parser.index >= len(parser.lines):
            break
        line = parser.lines[parser.index]
        parser.index += 1
        if not line.startswith("+"):
            raise ValueError(f"Invalid Add File Line: {line}")
        output.append(line[1:])

    return newline.join(output)


def _parse_update_diff(lines: list[str], input: str) -> ParsedUpdateDiff:
    parser = ParserState(lines=[*lines, END_PATCH])
    input_lines = input.split("\n")
    chunks: list[Chunk] = []
    cursor = 0

    while not _is_done(parser, END_SECTION_MARKERS):
        anchor = _read_str(parser, "@@ ")
        has_bare_anchor = (
            anchor == "" and parser.index < len(parser.lines) and parser.lines[parser.index] == "@@"
        )
        if has_bare_anchor:
            parser.index += 1

        if not (anchor or has_bare_anchor or cursor == 0):
            current_line = parser.lines[parser.index] if parser.index < len(parser.lines) else ""
            raise ValueError(f"Invalid Line:\n{current_line}")

        if anchor.strip():
            cursor = _advance_cursor_to_anchor(anchor, input_lines, cursor, parser)

        section = _read_section(parser.lines, parser.index)
        find_result = _find_context(input_lines, section.next_context, cursor, section.eof)
        if find_result.new_index == -1:
            ctx_text = "\n".join(section.next_context)
            if section.eof:
                raise ValueError(f"Invalid EOF Context {cursor}:\n{ctx_text}")
            raise ValueError(f"Invalid Context {cursor}:\n{ctx_text}")

        cursor = find_result.new_index + len(section.next_context)
        parser.fuzz += find_result.fuzz
        parser.index = section.end_index

        for ch in section.section_chunks:
            chunks.append(
                Chunk(
                    orig_index=ch.orig_index + find_result.new_index,
                    del_lines=list(ch.del_lines),
                    ins_lines=list(ch.ins_lines),
                )
            )

    return ParsedUpdateDiff(chunks=chunks, fuzz=parser.fuzz)


def _advance_cursor_to_anchor(
    anchor: str,
    input_lines: list[str],
    cursor: int,
    parser: ParserState,
) -> int:
    found = False

    if not any(line == anchor for line in input_lines[:cursor]):
        for i in range(cursor, len(input_lines)):
            if input_lines[i] == anchor:
                cursor = i + 1
                found = True
                break

    if not found and not any(line.strip() == anchor.strip() for line in input_lines[:cursor]):
        for i in range(cursor, len(input_lines)):
            if input_lines[i].strip() == anchor.strip():
                cursor = i + 1
                parser.fuzz += 1
                found = True
                break

    return cursor


def _read_section(lines: list[str], start_index: int) -> ReadSectionResult:
    context: list[str] = []
    del_lines: list[str] = []
    ins_lines: list[str] = []
    section_chunks: list[Chunk] = []
    mode: Literal["keep", "add", "delete"] = "keep"
    index = start_index
    orig_index = index

    while index < len(lines):
        raw = lines[index]
        if (
            raw.startswith("@@")
            or raw.startswith(END_PATCH)
            or raw.startswith("*** Update File:")
            or raw.startswith("*** Delete File:")
            or raw.startswith("*** Add File:")
            or raw.startswith(END_FILE)
        ):
            break
        if raw == "***":
            break
        if raw.startswith("***"):
            raise ValueError(f"Invalid Line: {raw}")

        index += 1
        last_mode = mode
        line = raw if raw else " "
        prefix = line[0]
        if prefix == "+":
            mode = "add"
        elif prefix == "-":
            mode = "delete"
        elif prefix == " ":
            mode = "keep"
        else:
            raise ValueError(f"Invalid Line: {line}")

        line_content = line[1:]
        switching_to_context = mode == "keep" and last_mode != mode
        if switching_to_context and (del_lines or ins_lines):
            section_chunks.append(
                Chunk(
                    orig_index=len(context) - len(del_lines),
                    del_lines=list(del_lines),
                    ins_lines=list(ins_lines),
                )
            )
            del_lines = []
            ins_lines = []

        if mode == "delete":
            del_lines.append(line_content)
            context.append(line_content)
        elif mode == "add":
            ins_lines.append(line_content)
        else:
            context.append(line_content)

    if del_lines or ins_lines:
        section_chunks.append(
            Chunk(
                orig_index=len(context) - len(del_lines),
                del_lines=list(del_lines),
                ins_lines=list(ins_lines),
            )
        )

    if index < len(lines) and lines[index] == END_FILE:
        return ReadSectionResult(context, section_chunks, index + 1, True)

    if index == orig_index:
        next_line = lines[index] if index < len(lines) else ""
        raise ValueError(f"Nothing in this section - index={index} {next_line}")

    return ReadSectionResult(context, section_chunks, index, False)


@dataclass
class ContextMatch:
    new_index: int
    fuzz: int


def _find_context(lines: list[str], context: list[str], start: int, eof: bool) -> ContextMatch:
    if eof:
        end_start = max(0, len(lines) - len(context))
        end_match = _find_context_core(lines, context, end_start)
        if end_match.new_index != -1:
            return end_match
        fallback = _find_context_core(lines, context, start)
        return ContextMatch(new_index=fallback.new_index, fuzz=fallback.fuzz + 10000)
    return _find_context_core(lines, context, start)


def _find_context_core(lines: list[str], context: list[str], start: int) -> ContextMatch:
    if not context:
        return ContextMatch(new_index=start, fuzz=0)

    for i in range(start, len(lines)):
        if _equals_slice(lines, context, i, lambda value: value):
            return ContextMatch(new_index=i, fuzz=0)
    for i in range(start, len(lines)):
        if _equals_slice(lines, context, i, lambda value: value.rstrip()):
            return ContextMatch(new_index=i, fuzz=1)
    for i in range(start, len(lines)):
        if _equals_slice(lines, context, i, lambda value: value.strip()):
            return ContextMatch(new_index=i, fuzz=100)

    return ContextMatch(new_index=-1, fuzz=0)


def _equals_slice(
    source: list[str], target: list[str], start: int, map_fn: Callable[[str], str]
) -> bool:
    if start + len(target) > len(source):
        return False
    for offset, target_value in enumerate(target):
        if map_fn(source[start + offset]) != map_fn(target_value):
            return False
    return True


def _apply_chunks(input: str, chunks: list[Chunk], newline: str) -> str:
    orig_lines = input.split("\n")
    dest_lines: list[str] = []
    cursor = 0

    for chunk in chunks:
        if chunk.orig_index > len(orig_lines):
            raise ValueError(
                f"applyDiff: chunk.origIndex {chunk.orig_index} > input length {len(orig_lines)}"
            )
        if cursor > chunk.orig_index:
            raise ValueError(
                f"applyDiff: overlapping chunk at {chunk.orig_index} (cursor {cursor})"
            )

        dest_lines.extend(orig_lines[cursor : chunk.orig_index])
        cursor = chunk.orig_index

        if chunk.ins_lines:
            dest_lines.extend(chunk.ins_lines)

        cursor += len(chunk.del_lines)

    dest_lines.extend(orig_lines[cursor:])
    return newline.join(dest_lines)


__all__ = ["apply_diff"]`
```
