---
name: explain-code
description: Explains code with visual diagrams and analogies. Use when explaining how code works, teaching about a codebase, or when the user asks "how does this work?"
---

If $ARGUMENTS is provided, use it as the file path, function name, or code reference to explain.
Focus the explanation on the specific function/class/module asked about — don't try to explain the entire file.

When explaining code, always include:

1. **Start with an analogy**: Compare the code to something from everyday life
2. **Draw a diagram**: Use Mermaid to show the flow, structure, or relationships. Pick the right type: `sequenceDiagram` for temporal flow, `flowchart` with `subgraph` + `classDef` for static structure. Render to PNG with `mmdc -i diagram.mmd -o diagram.png` and show the result.
3. **Walk through the code**: Explain step-by-step what happens
4. **Highlight a gotcha**: What's a common mistake or misconception?

Keep explanations conversational. For complex concepts, use multiple analogies.
