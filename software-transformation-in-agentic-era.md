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

Zhipu AutoGLM can simulate human to operate phone across multiple applicaions to finish specific task.

The provided features still are contrained by the software fixed logic, and can only deal with pre-designed scenarios.

## Pattern 2 - unlimited scenario, fixed logic

In this pattern, in fact, software doesn't change any. The agentic capability is added overlay via the agent for example OpenAI operator, or Google project mariner; both these two agents can handle tasks in web browser like navigation, fill form, etc.

Zhipu AutoGLM can simulate human to operate phone across multiple applicaions to finish specific task.

The provided features still are contrained by the software fixed logic, and can only deal with pre-designed scenarios.

[1] [Practices for Governing Agentic AI Systems](https://cdn.openai.com/papers/practices-for-governing-agentic-ai-systems.pdf)

[2] [Introducing Gemini 2.0: our new AI model for the agentic era](https://blog.google/technology/google-deepmind/google-gemini-ai-update-december-2024/#ceo-message)

[3] [NVIDIA CEO Jensen Huang Keynote at CES 2025](https://www.youtube.com/watch?v=k82RwXqZHY8)

[4] [The Anatomy of Agentic AI](https://dr-arsanjani.medium.com/the-anatomy-of-agentic-ai-0ae7d243d13c)

[5] [OpenAI Operator](https://openai.com/index/introducing-operator/)

[6] [Goole DeepMind Project Mariner](https://deepmind.google/technologies/project-mariner/)

[7] [AutoGLM: Autonomous Foundation Agents for GUIs](https://xiao9905.github.io/AutoGLM/)



(please note that all statements in the article is author's personal view, does not relect any view of his affiliation and the affilation's commercial interest. all references are public available resources)
