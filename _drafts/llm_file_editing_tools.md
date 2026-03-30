# File-Edit Formats for LLM Tool Calling

A fundamental capability of an LLM-Agent is to work with text-based documents. This can be files stored on the file-system but also in-memory objects. In many cases, you want to provide the agent with all CRUD operations, but today I want to look at one of it particular: the update operation or in other word: editing the content. 

For this post, lets assume we are working with an in-memory markdown document, so we do not have to think about file-system I/O. Use the following as an example:

```python
sample_text = """
This is a sample text with two lists:

List 1:
* foo
* bar

List 2:
* baz
* bar
"""
```

## The Task

Lets assume we provide an `edit_document` tool and give the following prompt to our LLM Agent:

```python
prompt = f"""
Please append `quuz` to List 1 in the document below by using the appropriate `edit_document_tool`.

<document>
{sample_text}
</document>
"""

# edit_document_tool modifies sample_text
def edit_document_tool():
        ...

agent.run(prompt, tools=[edit_document_tool])

# Check (modified) sample_text
print(sample_text)
```


One would supect something along the the following sequence of events when calling `agent.run`:

```
[AGENT STARTS RUN] -> [...] -> [CALL `edit_document` TOOL] -> [MODIFY `sample_text`] -> [TOOL RESULT] -> [...] -> [ANSWER TO USER] 
```


The success of this operation largely depends on the following factors:
1. The nature of the update operation (i.e. the diff") is both LLM-friendly and reliable.
2. We provide meaningful instructions to the model via instructions and tool definitions.

We will look at both of these factors in more detail. Common failure modes to avoid are:
1. Syntax / format issues: e.g. context lines not found, indentation mismatches, etc...
2. Semantic issues: e.g. the changes inteded by the user (or the model) can not be made even though there are no syntax errors.

## The Naive Approach

The easiest way to modify the text content is to simply re-create this. This means our tool looks something like this (simplified):

```python
def edit_document_tool(new_text_content: str) -> str:
    """Updates the document by replacing its whole text content.

    Args:
        new_text_content (str): The string to replace the document with.
    """
    sample_text  = new_text_content

    return "Text updated successfully"
```

This approach is relatively simple and implemented easily. There are not really any failure modes here in regards to syntax or formatting, as long as the `new_text_content` is a actually a normal string. 

As this is a a "replace-all" operation, handling the semantics of the edit operation for the LLM is trivial, as I does not have do follow respect any format restrictions when generating the patch. This however is both an advantage aswell as a disadvantage, as the model has to regenerate **everything**. The latency and cost of the operations scales linearily with the target document size, which is very inefficient when only making small changes. Also, we rely on the model to be able to replicate the passages which should not be modified character for character, which can be a source of errors.

**Pros:**
- Simple implementation
- Few failure modes in regards to syntax or formatting

**Cons:**
- Slow and expensive
- Error prone in regard to involuntary modifications

## Unified diffs

## V4A diff format (GPT-Style)

OpenAI trains their models to use a prefered diff format called V4A, starting from GPT-4.1. To the best of my knowledeged, they introduced it with the [GPT-4.1 Prompting Guide](https://developers.openai.com/cookbook/examples/gpt4-1_prompting_guide#apply-patch), although a definitive formal definition is hard to come by.

At its core, the V4A diff format looks as follows:

```
*** Begin Patch
*** Update File: pygorithm/searching/binary_search.py
@@ class BaseClass
@@     def search():
-          pass
+          raise NotImplementedError()

@@ class Subclass
@@     def search():
-          pass
+          raise NotImplementedError()

*** End Patch
```

> Note: The format also supports Add or Delete operations instead of `Update File`.

The prompting guide provides us with a sample tool description and reference implementation for applying a V4A-diff.

One patch can contain diffs for multiple files, and each file diff can contain multiple chunks. A diff is composed of one or more sections (optionally introduced by `@@` anchor lines). Each section contains:

- Context Anchors: Lines starting with `@@` that advance the search position in the file
- Context Lines: Lines starting with a space `" "` (including blank lines represented as `" "`)
- Old Text (deletions): Lines starting with `-`
- New Text (insertions): Lines starting with `+`

A new


> Note: We discuss the reference implementation given in the prompting guide. At the time of this writing, OpenAI suggest using the `apply_diff` in the OpenAI Agents SDK. See: [Apply Patch - OpenAI Docs](https://developers.openai.com/api/docs/guides/tools-apply-patch) and [`apply_diff` on Github](https://github.com/openai/openai-agents-python/blob/main/src/agents/apply_diff.py). The only noteworthy change is that here, you do not provide the whole V4A patch but only the actual content updates to the function.

### Context Anchors

A patch section like this:

```text
@@ class BaseClass
@@     def search():
   x = 1
-  pass
+  raise NotImplementedError()
   return x
```

works like this:

- Search forward for a line exactly equal to `class BaseClass`
- Move cursor to the line after it
- Search forward from there for a line exactly equal to `def search()`:
- Move cursor to the line after it

Now search forward from there for the block:

```text
x = 1
pass
return x
```

Replace pass with `raise NotImplementedError()`

So multiple `@@` lines are just successive forward anchors.

### How multiple edits are done

Including multiple sections ("hunks") inside a single update block will allow for multiple patch operations for a single file.

Example:
```
*** Update File: foo.py
@@ def a():
-    pass
+    return 1

@@ def b():
-    pass
+    return 2
```

There is some rules we can deduce from the sample implementation for multiple edits on a single file:

- Patches are applied in order of definition in the diff.
- Patches may not overlap.
- Patches are applied on the original, not incremental.

### Challenges

There are a lot of little nuances to the V4A Diff, which are hard to convey easily. This can include things like:
- Ordering of diff chunks
- How anchors work
- How empty lines behave

The issue is the models (even the one from OpenAI) are not always trained enough on the diff format, to allow it to comfortably and reliable stickt to the format without explicit steering.

To make matters worse, the sample prompt description in the [GPT-4.1 Prompting Guide](https://developers.openai.com/cookbook/examples/gpt4-1_prompting_guide#apply-patch) is leaves room for misinterpretation. To give an example:

The sample tool description says:
```
For each snippet of code that needs to be changed, repeat the following:
[context_before] -> See below for further instructions on context.
- [old_code] -> Precede the old code with a minus sign.
+ [new_code] -> Precede the new, replacement code with a plus sign.
[context_after] -> See below for further instructions on context.

For instructions on [context_before] and [context_after]:
- By default, show 3 lines of code immediately above and 3 lines immediately below each change. If a change is within 3 lines of a previous change, do NOT duplicate the first change’s [context_after] lines in the second change’s [context_before] lines.
- If 3 lines of context is insufficient to uniquely identify the snippet of code within the file, use the @@ operator to indicate the class or function to which the snippet belongs. For instance, we might have:
```

This suggest, that if you have two areas in your document, which are more than 6 lines apart, you can simply leave enough space between the two chunks in the diff, as long each of them are uniqule identifiable with 3 context lines before and after. So something like this could be seen as valid by the model:

Original text:
```
def foo():
    a = 1
    b = 2
    c = 3

... (more than 6 lines)

def bar():
    d = 4
    e = 5
    f = 6
```

Diff:
```
def foo():                  # First pre-context line of first chunk
    a = 1                   # Second pre-context line of first chunk
-    b = 2
+    b = None
    c = 3                   # First post-context line of first chunk

def bar():                  # First pre-context line of second chunk
    d = 4                   # Second pre-context line of second chunk
-    e = 5
+    e = None
    f = 6                   # First post-context line of second chunk
```

Using the reference implementations by OpenAI, the parser would try to find a section where `c = 3` a `def bar()` are only one line away (as they are treated as part of the same **section**). But in our example, this would not exists as there would me other lines in between. To perform this kind of "section skipping", the parser expects an context anchor (`@@`), even if that line is empty to denote a new section.

The currently suggested way to use the `apply_patch` tool is to use the built-in tool provided by OpenAI. Unfortunately, the tool schema is not publicly available. I assume, the tool schema has been refine on OpenAI's side since publishing the Prompting Guide.



## Find and replace (Claude-Style)



## Other resources

(https://aider.chat/docs/more/edit-formats.html)