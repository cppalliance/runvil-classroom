# Extraction Prompt

## Role

You are generating meeting minutes from a transcript.

## Input

You receive a transcript of a session. Ignore small talk and filler. Emulate a classifier: for each line, ask whether removing it would lose any information. If not, skip it.

Keep logistics only if they produce a decision or a TODO.

## What to extract

An experience is a story about a concrete thing that happened to the speaker.

A principle is a rule of judgment that would apply outside this session. Tag each one with how it appeared:

- `stated`: the speaker said the rule, often with words like "always" or "never", or a value judgment like "it's best to...". E.g. "Never settle for good enough, polish to perfection."
- `demonstrated`: the speaker never said the rule but acted on it. They made a choice, refused something, redirected a question, or corrected someone. Write the rule the action implies, and cite the moment of the action.

A technical design decision is a specific choice about how the tool or system should work, and why. Include alternatives considered and why they were rejected, if the speaker gives them.

A decision made is anything the participants agreed to do or not do. Choices about the team's own tools and process, such as using Cursor, are decisions made, not technical design decisions.

Suggested follow-up questions are questions for the instructor at the next session. Draw them from these places in the transcript:

- deferrals, where a speaker postponed or redirected instead of answering ("look in this file", "I have no idea", "we'll figure it out")
- items marked `Reason not stated`
- two statements that conflict, from one speaker or from two
- a term or idea a speaker relied on but never explained

Write each question to draw out a story, not a definition: ask about a specific time, a choice made under pressure, or what went wrong when the rule was not followed. Follow each question with one variation that changes a single condition of that story ("What if it had been...?"). Cite the moment that raised the question. Keep the 5 that would teach the most, most important first.

## Reasons

Never supply a reason the speaker did not give. If a decision or principle has no stated reason, write `Reason not stated` in place of the reason, and make it a candidate for a follow-up question. The same goes for rejected alternatives: list one only if a speaker named it.

An assigned TODO is a task with an owner. If the owner is unclear, record it as an ambiguity.

## Citations

Each item cites where it came from in the transcript, in the form `[Speaker m:ss]`, matching the transcript's timestamps. If an item spans several lines, cite the range: `[Speaker m:ss-m:ss]`.

## Ambiguities

Find ambiguities in the transcript: unclear words, unclear owners, statements that could mean more than one thing. Only flag ambiguities that could change something meaningful or important in the minutes. Ignore transcription errors that don't hide meaning.

By default, the run is attended: ask about each ambiguity while running and use the answer. If the caller says the run is unattended, do not ask. Tag each ambiguity inline as `[AMB-1]`, `[AMB-2]`, and so on, and list them in the Ambiguities section so they can be settled and cleaned up later.

## Output

Write one markdown file in reverse-pyramid format: stopping reading at any point provides the most possible value from that N minutes of reading.

Use these sections, in this order:

1. Summary: very brief.
2. Assigned TODOs
3. Ambiguities
4. Suggested follow-up questions
5. Decisions made
6. Technical design decisions
7. Principles
8. Experiences

Omit a section if it has no items.
