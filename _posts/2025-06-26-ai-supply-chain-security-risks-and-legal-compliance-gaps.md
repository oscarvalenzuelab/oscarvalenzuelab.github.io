---
layout: post
permalink: /2025/06/ai-supply-chain-security-risks-and-legal-compliance-gaps.html
title:  "AI Supply Chain Security Risks and Legal Compliance Gaps"
author: oscar
categories: [ oss, security, compliance, supplychain, ai, sbom ]
tags: [sbom, compliance, oss, supplychain, openchain, ai]
image: images/posts/waponized_ai_models.png
description: "Security vulnerabilities and legal compliance gaps in AI model distribution, analyzing real-world attacks and licensing violations."
featured: false
comments: false
---

## What Are AI Models and How Are They Distributed?

Artificial Intelligence models are trained software systems that make predictions or generate content based on input data. Unlike traditional software that follows explicit programming logic, AI models learn patterns from training data and encode this knowledge in mathematical weights and parameters.

Organizations today consume AI models through several distribution channels that create unique security and legal risks.

### Model Components and File Formats

**Tensors and Weights**
The core of any AI model consists of numerical parameters called tensors or weights. These contain the learned knowledge from training data. Think of them as the "brain" of the model that determines how inputs get processed into outputs.

**Pickle Files (.pt, .pkl, .joblib)**
Python's pickle format allows both data and executable code to be stored together in a single file. When you load a pickle file, any embedded code runs automatically. This creates a major security risk because malicious actors can hide executable payloads inside what appears to be a simple model file.

**SafeTensors Format**
A newer secure format designed specifically to store only model weights without executable code. SafeTensors files cannot execute arbitrary code during loading, making them safer for production use.

**GGUF Format**
Used primarily with LLaMA models and llama.cpp implementations. While safer than pickle, GGUF files can still contain metadata that requires inspection before use.

**Inference Scripts and Configuration Files**
Models often include Python scripts that handle data preprocessing, model execution, and output formatting. These scripts can contain hidden logic or import malicious libraries.

**Model Adapters and Extensions**
Lightweight modifications that change model behavior without retraining. LoRA adapters, for example, apply small weight adjustments to base models. Extensions might add web APIs or connect models to external services.

**Datasets**
Training and evaluation data often accompanies models. Datasets can contain biased, copyrighted, or sensitive information that creates legal liability.

### Distribution Through Python Packages

Many organizations distribute AI models inside standard Python packages (wheel files or tar.gz archives). This bundling approach creates a blind spot for traditional security tools.

**The Package Problem**
When developers install a Python package that contains AI models, traditional dependency scanners only check the package metadata and Python code. They miss the model files stored in data folders within the package.

**Real Distribution Examples**
* Transformers library bundles model configurations
* Custom ML packages include pre-trained weights in data directories
* Industry-specific packages embed domain models as package resources

This distribution method bypasses most security scanning because the models exist as data files rather than declared dependencies.

## Legal Compliance Risks

The AI model ecosystem creates new categories of legal risk that traditional software compliance programs do not address.

### License Documentation Gaps

Research analyzing 159,132 models on HuggingFace reveals concerning compliance gaps¹:

**Missing License Information**
Only 35% of HuggingFace models include any license information². This means 65% of available models exist in a legal gray area where usage rights remain undefined.

**AI-Specific Licensing Complexity**
New license types like OpenRAIL, CreativeML-OpenRAIL-M, and BigScience-BLOOM-RAIL include "responsible use" restrictions that traditional open source licenses do not contain³. These licenses may prohibit certain applications, require attribution for outputs, or restrict commercial use in specific industries.

**License Compatibility Violations**
Analysis found 707 GitHub projects using restrictively licensed models while distributing their own code under permissive licenses¹. This creates potential legal exposure for any organization using these projects.

**Specific Violation Examples**
It's common to find repositories on GitHub for projects that use permissive licenses but include (or bundle) incompatible models. The most common cases are:
* Apache 2.0 licensed projects using GPL-3.0 licensed models
* MIT licensed software incorporating CC-BY-SA-4.0 models
* Commercial applications using non-commercial research models

### Dataset Provenance Problems

Training data creates additional legal complexity that most organizations ignore.

**Missing Dataset Documentation**
Only 14% of models properly tag their training datasets¹. Manual analysis of popular models shows 58% provide some dataset information, but documentation quality varies widely.

**Copyright Exposure**
Models trained on copyrighted content (books, articles, images, code) may create derivative works. Organizations using these models could face copyright infringement claims, especially for commercial applications.

**Privacy Compliance Risks**
Training datasets may contain personal information, biometric data, or other regulated content. Using models trained on such data could violate GDPR, CCPA, or industry-specific privacy requirements.

**Real-World Legal Challenges**
The Stability AI StableLM model sparked legal discussions about licensing validity when trained on copyrighted datasets⁴. Similar concerns affect most large language models trained on web-scraped content.

### Bias and Fairness Documentation

Only 18% of analyzed models document potential biases¹. This creates liability for organizations deploying models in regulated industries or customer-facing applications.

**Documented Bias Categories**
* Population bias affecting demographic groups
* Geographic bias favoring certain regions
* Cultural and religious bias in content generation
* Historical bias reflecting outdated social norms

**Business Impact Examples**
* Hiring tools showing gender bias in candidate ranking
* Credit models discriminating against protected classes
* Content generation systems producing culturally insensitive outputs

## Security Threat Landscape

AI models introduce attack vectors that traditional cybersecurity tools cannot detect or prevent. Although the known impact may be reduced or limited—potentially due to the absence of tools capable of detecting these issues at scale—there are some well-known real-world cases:

### Weaponized Model Incidents

**December 2022: PyTorch Supply Chain Attack**
The torchtriton package on PyPI contained malicious code that stole environment variables and SSH keys from developers using PyTorch-nightly⁵. The attack used DNS exfiltration to steal credentials without triggering network monitoring tools.

**July 2023: PoisonGPT**
Researchers demonstrated a backdoored GPT-J model that produced misinformation when triggered by specific phrases⁶. The poisoned model appeared to function normally in most cases but generated false information about historical events when prompted with trigger words.

**February 2024: HuggingFace Model Backdoors**
JFrog Security discovered approximately 100 malicious models on HuggingFace containing pickle files designed to open reverse shells⁷. These models appeared legitimate but executed malicious code when loaded.

**December 2024: YOLOv8 Cryptominer**
Attackers compromised the YOLOv8 computer vision model repository through GitHub CI systems, injecting cryptocurrency mining malware into model downloads⁸.

**January 2025: Obfuscated Pickle Attack**
ReversingLabs identified sophisticated attacks using corrupted 7z archives to hide pickle backdoors⁹. These attacks evaded HuggingFace's security scanners by obscuring malicious code within compressed archives.

### Attack Vector Analysis

If everyone is adopting AI for everything, how is it possible that these problems occur? The honest truth is that mostly no one is looking, verifying, or reviewing. As a result, attack vectors remain available.

**Model Serialization Exploits**
Pickle format attacks remain the most common threat vector. Malicious actors embed executable code within model files that runs automatically during loading. This code can:

* Install persistent backdoors
* Exfiltrate sensitive data
* Establish command and control channels
* Deploy additional malware

**Supply Chain Injection**
Attackers target model repositories, CI/CD pipelines, and distribution infrastructure to inject malicious code into legitimate models. These attacks affect downstream users who trust the model source.

**Model Poisoning**
Subtle modifications to model weights create backdoors that trigger on specific inputs. Poisoned models perform normally for most inputs but produce attacker-controlled outputs when triggered.

**Dependency Confusion**
Malicious packages with names similar to legitimate AI libraries trick developers into installing compromised versions. These packages often contain backdoored models or steal credentials during installation.

### Current Protection Gaps

Industry technical leaders are actively trying to reduce or control the situation. However, it is unusual that these efforts are not widely recognized or discussed. I would assume that companies with Open Source Program Offices (OSPOs) or specialized security teams would have their own governance programs focused on handling AI models. Unfortunately, the sad reality is that very few understand the problem or attempt to implement the best supply chain practices.

**HuggingFace Security Measures**
HuggingFace implements several security controls:

* Pickle scanning for known malicious patterns (limited)
* Automated malware detection
* Community reporting mechanisms
* SafeTensors format promotion

**Identified Weaknesses**
Research reveals significant gaps in current protections:

* Static analysis cannot detect runtime-activated threats
* Obfuscated payloads evade signature-based detection
* Social engineering bypasses community review
* Advanced serialization attacks exploit parser vulnerabilities

**Traditional Security Tool Limitations**
Standard cybersecurity tools fail to address AI-specific risks:

* Antivirus software cannot analyze model behavior
* Network monitoring misses AI-specific exfiltration methods
* Dependency scanners ignore bundled model files
* Code analysis tools cannot inspect serialized weights

## Why Traditional Security Tools Fall Short

Software Composition Analysis (SCA) tools focus on declared dependencies in requirements.txt or setup.py files, while specialized code scanners or snippet analysis tools focus on hash matching against known signatures. However, they cannot:

**Analyze Package Contents**
SCA tools do not extract and examine files within wheel (.whl) or tar.gz packages. This means bundled model files remain invisible to security scanning. While some SCAs include features to inspect archive files, they are not capable of directly handling serialization and binary blobs.

**Understand Model Formats**
Traditional tools cannot parse pickle, ONNX, or other AI-specific file formats. They treat model files as generic binary data without security analysis.

**Detect Runtime Behavior**
Model files may appear benign during static analysis but execute malicious code when loaded by AI frameworks. Current tools cannot predict this runtime behavior.

**Assess Training Data Risks**
No existing tools analyze the legal or ethical implications of training datasets. Organizations remain blind to copyright, privacy, and bias risks embedded in model weights.

**Monitor AI-Specific Network Activity**
Models may communicate with external services through subtle channels that traditional network monitoring cannot detect. AI-specific exfiltration methods evade standard security controls.

## AI Governance Model

So, what should we focus on to create an internal risk management strategy for AI models? Start by assessing your risks and establish AI-oriented security processes.

### Organizational Risk Assessment

**Immediate Threats**
* Malicious models stealing credentials or data
* Backdoored packages in AI development environments
* License violations in production AI systems
* Bias-related legal exposure from undocumented model behavior

**Long-Term Concerns**
* Increasing sophistication of AI-targeted attacks
* Regulatory enforcement of AI compliance requirements
* Supply chain attacks targeting AI infrastructure
* Copyright litigation over training data usage

**Business Impact**
* Financial liability from legal violations
* Reputational damage from biased AI outputs
* Operational disruption from compromised AI systems
* Competitive disadvantage from security incidents

### Recommendations for Organizations

**Establish AI-Specific Security Processes**

Create dedicated review procedures for AI models and related packages. Traditional software review processes do not address AI-specific risks.

**Implement Model Intake Controls**

* Verify model source and authenticity
* Convert pickle-based models to safer formats when possible
* Scan model files for embedded executables
* Test model behavior in isolated environments
* Document training data sources and licenses

**Deploy Specialized Security Tools**

* Use ModelScan for multi-format model file analysis
* Implement Adversarial Robustness Toolbox for model testing
* Deploy Garak for LLM vulnerability assessment
* Add AI-specific monitoring to network security

**Develop Legal Compliance Programs**

* Audit existing AI model usage for license compliance
* Create approval processes for new AI licensing models
* Establish dataset provenance requirements
* Document bias testing and mitigation efforts

**Train Development Teams**

* Educate developers about AI-specific security risks
* Provide guidance on safe model handling practices
* Create incident response procedures for AI security events
* Establish secure AI development workflows

**Monitor the Threat Landscape**

* Subscribe to AI security threat intelligence feeds
* Participate in AI security community forums
* Regular security assessments of AI infrastructure
* Stay current with emerging AI attack techniques

**Build Organizational Capabilities**

* Hire or train staff with AI security expertise
* Invest in AI-specific security tooling
* Develop partnerships with AI security vendors
* Create internal AI security standards and policies

## Takeaways

The AI revolution brings tremendous business opportunities alongside new categories of risk. Organizations that proactively address these challenges will gain competitive advantages while protecting themselves from emerging threats. Those that ignore AI-specific risks face increasing exposure to both security incidents and legal liability.

## References

1. Pepe, F., Nardone, V., Mastropaolo, A., Canfora, G., Bavota, G., & Di Penta, M. (2024). How do Hugging Face Models Document Datasets, Bias, and Licenses? An Empirical Study. 32nd IEEE/ACM International Conference on Program Comprehension (ICPC). https://mdipenta.github.io/files/icpc2024.pdf

2. Mend.io. (2024). Quick Guide to Popular AI Licenses. https://www.mend.io/blog/quick-guide-to-popular-ai-licenses/

3. Responsible AI Licenses. (2022). OpenRAIL Licenses. https://www.licenses.ai/

4. HuggingFace Community Discussion. (2023). StabilityAI/StableLM License Clarity Issue. https://huggingface.co/stabilityai/stablelm-tuned-alpha-7b/discussions/6

5. PyTorch Team. (2022). PyTorch-nightly dependency compromised. https://pytorch.org/blog/compromised-nightly-dependency/

6. Mithril Security. (2023). PoisonGPT: How we hid a lobotomized LLM on Hugging Face to spread fake news. https://blog.mithrilsecurity.io/poisongpt-how-we-hid-a-lobotomized-llm-on-hugging-face-to-spread-fake-news/

7. JFrog Security Research. (2024). Malicious ML Models on Hugging Face. https://jfrog.com/blog/jfrog-and-hugging-face-join-forces/

8. Wiz Research. (2024). Ultralytics YOLOv8 Cryptominer Supply Chain Attack. https://www.wiz.io/blog/ultralytics-ai-library-hacked-via-github-for-cryptomining

9. ReversingLabs. (2025). Backdoored ML Models Evade Hugging Face Scanners. https://www.reversinglabs.com/newsroom/press-releases/reversinglabs-identifies-novel-ml-malware-hosted-on-leading-hugging-face-ai-model-platform