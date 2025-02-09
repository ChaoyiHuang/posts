# Software Transformation in Agentic Era

Agentic AI systems - AI systems that can pursue complex goals with limited direct supervision - is a new concept defined first in the paper "Practices for Governing Agentic AI Systems" by OpenAI, which was published on Dec.14, 2023.

"Agentic", "Agentic AI", "Agentic AI systems" have rapdily become buzzwords, though AI agent is a concept appeared decades ago.

Dec 11, 2024, when Gemini 2.0 was released, "our new AI model for the agentic era" claimed by Google.

Jan.6, 2025, "The age of Agentic AI is here", in Nvidia CEO Jensen Huang keynote at CES.

The core of Agentic AI systems is the reasoning/planning/tools-use capabilities etc powered by the breakthough LLM (including multi-modal large model) AI technologies.

the article "The Anatomy of Agentic AI" provides a comprehensive understanding of the technical point of view of Agentic AI systems.

Software is being reshaped in agentic era. There are several implementation patterns, as illustrated in the following picture.

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

## Pattern 2 - agentic inside, frontend

In pattern 2, the software interactive interfaces (mostly the frontend part) is changed to natrual language, or multi-modal ways (text, voice, vision etc), but the backend part keep the same.

It's open for scenario what the end user want to do, the flexibility to handle various scenarios is what the value added by agentic AI systems, powered by LLM/VLM etc.

But the features are still constrained by the backend which is traditional programmed software, what API is provided is what being exposed, nothing more.

The devops agent under AWS bedrock agent samples[8] is a good example for this pattern. The supervisor agent can act as intent recognition, route and orchestrate github agent and grafana agent to handle the task. While granfana agent can use tool(lambda function) to call grafana, what task can be completed is relying on the API provided by grafana, so as to github agent. 

## Pattern 3 - agentic inside, backend

This pattern will remove the programmed code rule constraints in service logic. Except the data schema, (either structured or unstructured), all service logic can be dynamicly generated according to requests, thus set no limitation on usage scenario, open scenario plus open logic.

The data schema is just to establish the boundary that the problem domain the agent should focus on.

Smolagents provides a very good example on this pattern. The code for two tables could refer to attached [python script](attachments/text_to_sql.py), you have to replace your own huggingface token to run the script. Different service logic code will be dynamicly generated to address different user input, of course limited on the data schema defined domain.

## Pattern 4 - agentic native

Currently there are lots of low code or no code agent build platform, to create such kind of general purpose domain agents.

On the other hand, it may be purpose built agent, for example Tesla FSD, with continous online/offline training, with self evolution capabilities. 

For vertical industry, It may require huge mounts of data to train specific model and serve as the agent, in most cases, it may not be a good deal in ROI.

## References

[1] [Practices for Governing Agentic AI Systems](https://cdn.openai.com/papers/practices-for-governing-agentic-ai-systems.pdf)

[2] [Introducing Gemini 2.0: our new AI model for the agentic era](https://blog.google/technology/google-deepmind/google-gemini-ai-update-december-2024/#ceo-message)

[3] [NVIDIA CEO Jensen Huang Keynote at CES 2025](https://www.youtube.com/watch?v=k82RwXqZHY8)

[4] [The Anatomy of Agentic AI](https://dr-arsanjani.medium.com/the-anatomy-of-agentic-ai-0ae7d243d13c)

[5] [OpenAI Operator](https://openai.com/index/introducing-operator/)

[6] [Goole DeepMind Project Mariner](https://deepmind.google/technologies/project-mariner/)

[7] [AutoGLM: Autonomous Foundation Agents for GUIs](https://xiao9905.github.io/AutoGLM/)

[8] [amazon-bedrock-agent-samples-devops-agent](https://github.com/awslabs/amazon-bedrock-agent-samples/tree/main/examples/multi_agent_collaboration/devops_agent)

[9] [smolagents](https://github.com/huggingface/smolagents)

[10] [smolagents Text to SQL example](https://huggingface.co/docs/smolagents/examples/text_to_sql)

(please note that all statements in the article is author's personal view, does not reflect any view of his affiliation and the affilation's commercial interest. all references are public available resources)
