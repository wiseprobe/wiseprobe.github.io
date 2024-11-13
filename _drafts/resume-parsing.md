---
layout: post
title: Parsing Resumes With Local LLMs
categories: [Blog]
tags: [technology,llm,generative ai,information extraction]
---

Resume-parsing is a great use case for generative AI. In this post, we demonstrate how local LLMs like [Meta's Llama-3.1](https://ai.meta.com/blog/meta-llama-3-1/) can extract useful information from resumes to make them more findable. We will use the [OnPrem.LLM](https://amaiya.github.io/onprem/) package, a handy Python toolkit for working with LLMs.

By default [OnPrem.LLM](https://amaiya.github.io/onprem/) uses the [Llama-cpp -python](https://github.com/abetlen/llama-cpp-python) under the hood. However, OnPrem.LLM can be used with a number of local LLM tools like [Ollama and vLLM](https://amaiya.github.io/onprem/#connecting-to-llms-served-through-rest-apis).



## STEP 1: Load the Llama 3.1 Model

The choice of model is important, as certain local LLMs struggle with this task.  For instance, [Mistral-7B](https://huggingface.co/mistralai/Mistral-7B-Instruct-v0.3) doesn't seem to perform that well on resume-parsing tasks in our experiments. By contrast, the [Llama-3.1-8B model](https://huggingface.co/meta-llama/Llama-3.1-8B) from Meta performs quite nicely.

Llama.cpp  (and, by extension, OnPrem.LLM) requires models in GGUF format. There are different GGUF instances of this Llama-3.1-8B on the Hugging Face model hub, and we will use the [one from LM Studio](https://huggingface.co/lmstudio-community/Meta-Llama-3.1-8B-Instruct-GGUF). When selecting a model that is different than the default ones in OnPrem.LLM, it is important to inspect the model’s home page and identify the correct prompt format. The prompt format for this model is located [here](https://huggingface.co/lmstudio-community/Meta-Llama-3.1-8B-Instruct-GGUF#prompt-template), and we will supply it directly to the LLM constructor along with the URL to the specific model file we want (i.e., `Meta-Llama-3.1-8B-Instruct-Q4_K_M`.gguf). We will offload layers to our GPU(s) to speed up inference using the `n_gpu_layers` parameter. (For more information on GPU acceleration, see here.) For the purposes of this notebook, we also supply temperature=0 so that there is no variability in outputs. You can increase this value for more creativity in the outputs. Note that you can change the system prompt (i.e., “You are a super-intelligent helpful assistant…”) to fit your needs.


When using local LLMs, OnPrem.LLM requires models in GGUF format. In this example we will load a GGUF LLama-3.1 model from LM Studio.

```python
from onprem import LLM
import os
prompt_template = """<|start_header_id|>system<|end_header_id|>

You are a super-intelligent helpful assistant that executes instructions.<|eot_id|><|start_header_id|>user<|end_header_id|>

{prompt}<|eot_id|><|start_header_id|>assistant<|end_header_id|>
"""

llm = LLM(model_url='https://huggingface.co/lmstudio-community/Meta-Llama-3.1-8B-Instruct-GGUF/resolve/main/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf',
          prompt_template= prompt_template,
          n_gpu_layers=-1,
          temperature=0,
          verbose=False)
```

## STEP 2: Download a Resume

We will use my CV and consider the first page as the "resume" in this example. Type this at a command prompt to downlaod the CV or try it with your own resume.

```sh
!wget https://arun.maiya.net/asmcv.pdf -O /tmp/cv.pdf
```

## STEP 3: Load Text from the PDF Resume

The PDF we just downloaded is in PDF format, so we'll extract the text from it using the `load_single_document` function in OnPrem.LLM.

```python
from onprem.ingest import load_single_document
docs = load_single_document('/tmp/cv.pdf')
resume_text = docs[0].page_content # we'll only consider the first page of CV as "resume"
```


## STEP 4: Construct a Prompt for Resume-Parsing

We will develop a prompt that extracts information from resumes as a JSON string. We will include a placeholder for the resume text called `--RESUMETXT--`, which will be replaced with whatever resume we are anlayzing when we execute the prompt.

```python
prompt = """
Analyze the resume below and extract the relevant details. Format the response in JSON according to the specified structure below. 
Only return the JSON response, with no additional text or explanations.

Ensure to:
- Format the full name in proper case.
- Remove any spaces and country code from the contact number.
- Format dates as "dd-mm-yyyy" if given in a more complex format, or retain the year if only the year is present.
- Do not make up a phone number. 
- Extract only the first two jobs for Work Experience.

Use the following JSON structure:

`` ```json ``
{
 "Personal Information": {
    "Name": " ",
    "Contact Number": " ",
    "Address": " ",
    "Email": " ",
    "Date of Birth": " "
  },
  "Education": [
    {
      "Degree": " ",
      "Institution": " ",
      "Year": " "
    },
    // Additional educational qualifications in a similar format
  ],
  "Work Experience": [
    {
      "Position": " ",
      "Organization": " ",
      "Duration": " ",
      "Responsibilities": " "
    },
    // Additional work experiences in a similar format
  ],
  "Skills": [
  {
    "Skills": " ", // e.g., Python, R, Java, statistics, quantitative psychology, applied mathematics, machine learning, gel electrophoresis
  },
  // A list of skills or fields that the person has experience with
  ],
}
`` ``` ``

Here is the text of the resume:

---RESUMETXT---
"""
```

## STEP 5: Parse a Resume

Finally, we will insert the text of the resume into the prompt and send it to the LLM.

```python
output = llm.prompt(prompt.replace('---RESUMETXT---', resume_text))
```

The extracted information as a JSON string is:

```json
{
 "Personal Information": {
    "Name": "Arun S. Maiya",
    "Contact Number": "",
    "Address": "",
    "Email": "arun@maiya.net",
    "Date of Birth": ""
  },
  "Education": [
    {
      "Degree": "Ph.D.",
      "Institution": "University of Illinois at Chicago",
      "Year": " "
    },
    {
      "Degree": "M.S.",
      "Institution": "DePaul University",
      "Year": " "
    },
    {
      "Degree": "B.S.",
      "Institution": "University of Illinois at Urbana-Champaign",
      "Year": " "
    }
  ],
  "Work Experience": [
    {
      "Position": "Research Leader",
      "Organization": "Institute for Defense Analyses – Alexandria, VA USA",
      "Duration": "2011-Present",
      "Responsibilities": ""
    },
    {
      "Position": "Researcher",
      "Organization": "University of Illinois at Chicago",
      "Duration": "2007-2011",
      "Responsibilities": ""
    },
  ],
  "Skills": [
    {
      "Skills": "applied machine learning, data science, natural language processing (NLP), network science, computer vision"
    }
  ]
}
```



You can convert the JSON string to a Python dictionary as follows:

```python
import json
d = json.loads(output)
d.keys()
# dict_keys(['Personal Information', 'Education', 'Work Experience', 'Skills'])


```


For more information, see the [OnPrem.LLM documentation](https://amaiya.github.io/onprem/) and give it a star find it helpful.


