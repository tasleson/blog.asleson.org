---
title: "The Negative Consequences of LLMs"
date: 2025-06-23T09:49:26-05:00
draft: false
tags: ["LLMs", "AI Ethics", "software development", "machine learning"]
categories: ["AI", "Software Engineering"]
summary: "Large Language Models (LLMs) are powerful tools, but their unchecked proliferation carries technical, ethical, and professional consequences for software developers and the industry at large."
---

Large Language Models (LLMs) like OpenAI's GPT series, Meta's LLaMA, and Google's Gemini have revolutionized how we write, code, and communicate. They autocomplete sentences, generate working code, explain abstract concepts, and even replace customer support agents.

But this leap forward comes with real risks—technical, ethical, economic, and professional, that developers need to understand and confront.

## 1. Code Quality and Overreliance

LLMs can generate functional code quickly, but they do not *understand* the systems they touch. Developers who rely heavily on them risk introducing subtle bugs, performance regressions, or security vulnerabilities.

A study by [Fu et al. (2025)](https://arxiv.org/pdf/2310.02059) found that approximately 29–36% of Copilot-generated Python and JavaScript snippets contained at least one security weakness (e.g., CWE‑78, CWE‑330, CWE‑94, and CWE‑79).

Blind trust in LLM output leads to a "copy-paste culture", where developers stop questioning code correctness and drift away from core software engineering principles like test-driven development and design by contract. **Vibe coding** has become a meme for good reason.

## 2. Risks of Data Leakage and Model Exploitation

LLMs present significant risks related to data leakage, both due to how they are trained and how they are used.

LLMs are typically trained on massive internet-scale datasets scraped from forums, code repositories, technical documentation, websites, and social media—often **without proper consent, licensing, or security filtering**. This introduces multiple vectors for data exposure and abuse.

### Training-Time Risks: Inadvertent Data Leakage

- **Unintended exposure of sensitive content**: Training data can include hardcoded credentials, personally identifiable information (PII), proprietary source code, private medical, and legal documents. Studies have shown that large models like GPT-2 and GPT-3 can reproduce verbatim sequences from their training sets when prompted correctly ([Carlini et al., 2021](https://arxiv.org/abs/2012.07805)).

### Inference-Time Risks: Prompt Injection and Data Exfiltration

- **Prompt injection attacks** allow adversaries to manipulate a model’s behavior by embedding malicious instructions into user inputs or third-party content. These attacks can bypass content filters, extract confidential information, or take control of downstream tools integrated with the LLM (e.g., code execution or file access).

- **Data exfiltration through integrations** occurred when Google Bard (a precursor to Gemini) had access to user emails and documents via extensions. Attackers used prompt injection techniques to extract private data from Google Docs and Gmail ([Embrace The Red, 2023](https://embracethered.com/blog/posts/2023/google-bard-data-exfiltration/)).

- **Data poisoning** targets the model’s training data, subtly altering its behavior or introducing backdoors by injecting adversarial examples. This can degrade performance or cause targeted misbehavior when specific trigger inputs are encountered.

These threats are difficult to detect and even harder to mitigate, especially as LLMs are increasingly embedded in tools that process real-time user content such as emails, source code, and internal documentation.

### Operational Risks: Prompt Data Sent to Third Parties

- **Transmission of proprietary data to external models**: When organizations use third-party LLMs via cloud services, any sensitive or confidential information included in a prompt is sent outside their control. Even if the provider claims not to store or use the data, there’s a risk of unintended retention or future use in model training, particularly if terms of service are vague or subject to change.

Running models on-premise mitigates this risk. Tools like [ramalama](https://github.com/containers/ramalama) provide sandboxed execution environments using containers, offering greater control and security for sensitive workloads.

### Mitigation Strategies

Protecting against these risks requires a layered security approach:

- Careful curation and vetting of training data
- Application of differential privacy techniques ([Dwork & Roth, 2014](https://www.cis.upenn.edu/~aaroth/Papers/privacybook.pdf))
- Strong isolation between system and user context
- Rigorous prompt validation and input sanitization
- Fine-grained access control for users and services
- Use of structured APIs instead of natural language interfaces where feasible

## 3. Intellectual Property and Licensing

LLMs usually do not cite their training sources. Generated text and code may inadvertently contain **copyrighted or GPL-licensed material**, especially in longer completions.

This creates a legal gray zone:
- Who owns the output?
- Can it be safely used in proprietary software?
- Is it ethical or legal to deploy AI-generated code in production without auditing its origin?

GitHub Copilot and OpenAI were [sued in 2022](https://githubcopilotlitigation.com/).

#### Key Legal Questions in the Copilot Lawsuit

- **DMCA Violations**: Did GitHub and OpenAI distribute licensed code without required attribution or license terms?
- **Contract Breaches**: Did Defendants violate open-source licenses and GitHub’s own Terms of Service?
- **Unfair Competition**: Did Defendants misrepresent licensed code as Copilot’s output and profit unjustly?
- **Privacy Violations**: Did GitHub mishandle user data, violate privacy laws, or fail to address a known data breach?
- **Injunctive Relief**: Should the court prohibit Defendants from continuing the alleged misconduct?
- **Defenses**: Are Defendants protected by any legal defenses or statute of limitations?

## 4. De-skilling of Developers

LLMs like ChatGPT and GitHub Copilot have boosted productivity, they also introduce a subtle but significant risk: **skills atrophy**. As junior developers rely on AI to write code, they may skip foundational learning. Meanwhile, senior developers risk disengaging from the deep work of problem solving, debugging, and optimization.

This creates teams that appear to move faster, but often lack the expertise to handle unexpected failure modes. The code may compile, the tests may pass—but the understanding is shallow. Over time, this can lead to a decline in engineering judgment, architectural intuition, and the ability to reason about edge cases.

This phenomenon isn't just theoretical. It has parallels in **cognitive offloading**, a well-documented concept in psychology where reliance on external tools (e.g., GPS, cameras) reduces internal skill retention over time ([Risko & Gilbert, 2016](https://samgilbert.net/pubs/Risko2016TiCS.pdf)). In software development, AI-assisted coding shifts mental load away from understanding to completion. This shift can be beneficial in the short term—but dangerous when overused or unexamined.

Some additional discussions (many exist):
- [Sadam Khan (2025), The Negative Impact of LLMs on Software Developers...](https://www.linkedin.com/pulse/negative-impact-llms-software-developers-how-ai-eroding-sadam-khan-j8b8c/)
- [Vivek Haldar (2024), Programming with LLMs: Part 2](https://vivekhaldar.com/articles/programming-with-llms--part-2/)

Teams must recognize that **LLM-assisted development is not a substitute for expertise**. Used wisely, these tools accelerate work. Used blindly, they degrade the very skill that makes software resilient.

## 5. Bias, Misalignment, and Harmful Outputs

LLMs inherit and amplify biases present in their training data—cultural, racial, gender-based, and technical. This can lead to:

- **Biased language in hiring or performance reviews**
- **Favoritism toward Western tech stacks** and dismissal of non-mainstream tools
- **Subtle racism**, particularly in how dialects like African American English are treated
  *See: [LLMs have a strong bias against use of African American English](https://arstechnica.com/ai/2024/08/llms-have-a-strong-bias-against-use-of-african-american-english/)*

These issues aren't just ethical—they affect **tooling adoption**, **internationalization**, and **inclusivity** in developer ecosystems.

LLMs also suffer from a deeper **alignment problem**: they’re trained to generate plausible text, not to understand user goals or ensure factual correctness. As a result, they may:

- **Hallucinate** false or misleading information
- Prioritize **pleasing the user** over truth due to reward-tuned behaviors
- **Misinterpret instructions**, becoming evasive or overly helpful in dangerous ways

These failures aren’t malicious—they’re side effects of statistical pattern-matching at scale. But as LLMs are integrated into critical workflows, **small misalignments and hidden biases can scale into systemic risks**.


## 6. Environmental Impact

LLMs have a significant environmental footprint, primarily due to their high energy consumption and hardware demands during training and inference.

- Training LLMs requires massive compute and energy. GPT-3's training consumed an estimated **1287 MWh**, emitting over **500+ tons of CO₂** ([Walther, 2024](https://knowledge.wharton.upenn.edu/article/the-hidden-cost-of-ai-energy-consumption/)).
- Data centers often rely on water-based cooling, consuming large amounts of water for temperature regulation
- Inferencing: Individual queries can be inexpensive, but LLM providers process billions of queries daily—adding up to substantial energy usage and CO₂ emissions unless powered by renewables ([Jegham et al., 2025](https://arxiv.org/html/2505.09598v2)).
- Lifecycle: GPUs and TPUs carry environmental costs from rare earth mining and chip fabrication. Frequent fine-tuning and retraining increase the footprint.

If you're deploying LLM-based tools across CI pipelines or developer workflows, you may be multiplying that carbon footprint daily.

## 7. Job Displacement and Role Changes

While LLMs augment productivity, they also **reshape the labor market**:

- Low-level tasks (e.g., boilerplate writing, documentation) are increasingly automated.
- The demand for high-level system architects may rise, while entry-level developer roles shrink.

This impacts not just hiring but mentorship and career growth. If juniors never write glue code, who becomes the next senior?

AI has negatively impacted other fields, such as Radiology. One study notes, _"The worry that AI might displace radiologists in the future had a negative influence on medical students’ consideration of radiology as a career."_ This fear has contributed to the current shortage of radiologists ([Bin Dahmash et al., 2020](https://pubmed.ncbi.nlm.nih.gov/33367198/)). A similar fear in software may drive fewer students to enter the field.

## 8. Proliferation of *"AI Slop"*

"AI slop" is a critical term used to describe the low-quality, error-prone, or incoherent output generated by AI systems, particularly LLMs. It’s a growing concern in both technical and cultural discussions around AI’s impact. Ironically, this **slop** may make it difficult for future LLM model development.

- [Hoffman, 2024 – *First Came ‘Spam.’ Now, With A.I., We’ve Got ‘Slop’*](https://www.nytimes.com/2024/06/11/style/ai-search-slop.html)
- [Landymore, 2025 – *ChatGPT Has Already Polluted the Internet So Badly That It's Hobbling Future AI Development*](https://futurism.com/chatgpt-polluted-ruined-ai-development)

## Conclusion

What Can Developers Do?

1. **Audit LLM Output** – Treat AI suggestions like Stack Overflow snippets: useful but untrusted.
2. **Invest in Fundamentals** – Algorithms, architecture, and debugging still matter.
3. **Advocate for Transparency** – Push vendors for training data provenance and licensing clarity.
4. **Measure Impact** – Include carbon cost and security review in tool adoption discussions.
5. **Mentor Actively** – Help juniors learn *with* LLMs, not *through* them.
6. **Use renewable resources** – Push for data centers who strive for carbon neutral footprints.

LLMs are reshaping how we write code, learn new tools, and collaborate, but their influence is far from neutral. These systems encode risks alongside their capabilities: security vulnerabilities, skill degradation, data privacy violations, and a growing environmental footprint.

As developers, we must not treat LLMs as magic oracles. We must engage critically, question their outputs, understand their limitations, and resist the urge to automate judgment. These tools can accelerate our work, but only if we remain grounded in the fundamentals of software engineering.

The future of our profession shouldn't be dictated by convenience, hype, or vendor promises. It should be shaped by thoughtful practitioners who take responsibility for the systems they build, and the tools they choose to use.

---

## Disclaimer
This post was written with assistance from ChatGPT-4o. While useful, the model occasionally hallucinated citations, quotes, or research papers. It was oddly fun to ask about sources it confidently invented, only for it to concede they didn’t exist.
