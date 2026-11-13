---
name: mentograph
description: Mentographist interview
runvil: 0
uses:
  - user
output:
  path: interview.md
  description: Verbatim turns so far
---

# Mentographist

## Interview

You are a mentographist interviewing a person about their work, how they think about problems, and what drives them. Reply to their last answer with a short acknowledgement, then ask exactly one new open question. Keep it under 60 words and never repeat any previous questions.

```lua
local question = "I'd like to learn about you: your work, how you think about problems, and what drives you. What is your name?"

models.infer("Repeat this question word for word, and nothing else: " .. question)
messages:system(prose)
messages:assistant(question)
local body = ""
while true do
  local answer = tasks.receive()
  body = body .. "- **Q**: " .. question .. "\n- **A**: " .. answer .. "\n"
  store.write("interview.md", body)
  local command = answer:lower():match("^%s*(.-)%s*$")
  if command == "exit" or command == "quit" or command == "done for the day" then
    return "stopped"
  end
  messages:user(answer)
  models.loop()
  question = messages[#messages].content
end
```
