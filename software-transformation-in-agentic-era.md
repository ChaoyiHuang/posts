# software transformation in agentic era

Agentic AI systems - AI systems that can pursue complex goals with limited direct supervision - is a new concept defined first in the paper "Practices for Governing Agentic AI Systems" by OpenAI, which was published on Dec.14, 2023.

"Agentic", "Agentic AI", "Agentic AI systems" have rapdily become buzzwords, though AI agent is a concept defined decades ago.

Dec 11, 2024, when Gemini 2.0 was released, "our new AI model for the agentic era" was claimed by Google.

Jan.6, 2025, "The age of Agentic AI is here", in Nvidia CEO Jensen Huang keynote at CES.

The core of Agentic AI systems is the reasoning/planning/tools-use capabilities etc introduced by the breakthough LLM (including multi-modal large model) AI technologies.

the article "The Anatomy of Agentic AI" provides a comprehensive understanding of the technical point of view of Agentic AI systems.

Software is being reshaped in agentic era greatly. There are several implementation patterns, as illustrated in the following picture.

![](attachments/agentic-software.png?raw=true)

## Pattern 0 - rule based software

Pattern 0 does not include any agentic capability, it's traditional software, usually a big software system is consisted of client-side frontend and server-side backend, some software may only provide frontend part, or backend part.

The interactive method to these software systems usually involve keyboard, mouse and/or touch screen.

No matter backend or front end, what the software can do is pre-determined by the source code, i.e, the programmed code rules limit the features. 

Characteristic: constrained scenrio, fix logic.

## Pattern 1 - agentic overlay

In this pattern, in fact, software doesn't change any. The agentic capability is added overlay via the agent for example OpenAI operator, or Google project mariner; both these two agents can handle tasks in web browser like navigation, fill form, etc.

