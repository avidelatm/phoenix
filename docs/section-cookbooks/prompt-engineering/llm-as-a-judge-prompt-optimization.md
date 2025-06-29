# LLM as a Judge Prompt Optimization

{% embed url="https://youtu.be/pvef59pEmvo?si=exraZynpgczBAAQb" %}

## **LLM as a Judge**

An LLM as a Judge refers to using an LLM as a tool for evaluating and scoring responses based on predefined criteria.&#x20;

While LLMs are powerful tools for evaluation, their performance can be inconsistent. Factors like ambiguity in the prompt, biases in the model, or a lack of clear guidelines can lead to unreliable results. By fine-tuning your LLM as a Judveprompts, you can improve the model's consistency, fairness, and accuracy, ensuring it delivers more reliable evaluations.

In this tutorial, you will:

* Generate an LLM as a Judge evaluation prompt and test it against a datset
* Learn about various optimization techniques to improve the template, measuring accuracy at each step using Phoenix evaluations
* Understand how to apply these techniques together for better evaluation across your specific use cases

{% embed url="https://colab.research.google.com/github/Arize-ai/phoenix/blob/3e717b97a73f4fa0fb2f2ca1cd3d2911497a65c8/tutorials/evals/optimizing_llm_as_a_judge_prompts.ipynb#L4" %}

## **Set Up Dependencies and Keys**

```python
!pip install -q "arize-phoenix>=8.0.0" datasets openinference-instrumentation-openai
```

Next you need to connect to Phoenix. The code below will connect you to a Phoenix Cloud instance. You can also [connect to a self-hosted Phoenix instance](https://arize.com/docs/phoenix/deployment) if you'd prefer.

```python
import os
from getpass import getpass

os.environ["PHOENIX_COLLECTOR_ENDPOINT"] = "https://app.phoenix.arize.com"
if not os.environ.get("PHOENIX_CLIENT_HEADERS"):
    os.environ["PHOENIX_CLIENT_HEADERS"] = "api_key=" + getpass("Enter your Phoenix API key: ")

if not os.environ.get("OPENAI_API_KEY"):
    os.environ["OPENAI_API_KEY"] = getpass("Enter your OpenAI API key: ")
```

## **Load Dataset into Phoenix**

Phoenix offers many [pre-built evaluation templates](https://arize.com/docs/phoenix/evaluation/concepts-evals/evaluation) for LLM as a Judge, but often, you may need to build a custom evaluator for specific use cases.

In this tutorial, we will focus on creating an LLM as a Judge prompt designed to assess empathy and emotional intelligence in chatbot responses. This is especially useful for use cases like mental health chatbots or customer support interactions.

We will start by loading a dataset containing 30 chatbot responses, each with a score for empathy and emotional intelligence (out of 10). Throughout the tutorial, we’ll use our prompt to evaluate these responses and compare the output to the ground-truth labels. This will allow us to assess how well our prompt performs.

```python
from datasets import load_dataset

ds = load_dataset("syeddula/empathy_scores")["test"]
ds = ds.to_pandas()
ds.head()

import uuid

import phoenix as px

unique_id = uuid.uuid4()

# Upload the dataset to Phoenix
dataset = px.Client().upload_dataset(
    dataframe=ds,
    input_keys=["AI_Response", "EI_Empathy_Score"],
    output_keys=["EI_Empathy_Score"],
    dataset_name=f"empathy-{unique_id}",
)
```

## **Generate LLM as a Judge Template using Meta Prompting**

Before iterating on our template, we need to establish a prompt. Running the cell below will generate an LLM as a Judge prompt specifically for evaluating empathy and emotional intelligence. When generating this template, we emphasize:

* Picking evaluation criteria (e.g., empathy, emotional support, emotional intelligence).
* Defining a clear scoring system (1-10 scale with defined descriptions).
* Setting response formatting guidelines for clarity and consistency.
* Including an explanation for why the LLM selects a given score.

```python
from openai import OpenAI

client = OpenAI()


def generate_eval_template():
    meta_prompt = """
    You are an expert in AI evaluation and emotional intelligence assessment. Your task is to create a structured evaluation template for assessing the emotional intelligence and empathy of AI responses to user inputs.

    ### Task Overview:
    Generate a detailed evaluation template that measures the AI’s ability to recognize user emotions, respond empathetically, and provide emotionally appropriate responses. The template should:
    - Include 3 to 5 distinct evaluation criteria that assess different aspects of emotional intelligence.
    - Define a scoring system on a scale of 1 to 10, ensuring a broad distribution of scores across different responses.
    - Provide clear, tiered guidelines for assigning scores, distinguishing weak, average, and strong performance.
    - Include a justification section requiring evaluators to explain the assigned score with specific examples.
    - Ensure the scoring rubric considers complexity and edge cases, preventing generic or uniform scores.

    ### Format:
    Return the evaluation template as plain text, structured with headings, criteria, and a detailed scoring rubric. The template should be easy to follow and apply to real-world datasets.

    ### Scoring Guidelines:
    - The scoring system must be on a **scale of 1 to 10** and encourage a full range of scores.
    - Differentiate between strong, average, and weak responses using specific, well-defined levels.
    - Require evaluators to justify scores

    Do not include any concluding remarks such as 'End of Template' or similar statements. The template should end naturally after the final section.

    """

    try:
        response = client.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": meta_prompt}],
            temperature=0.9,  # High temperature for more creativity
        )

        return response.choices[0].message.content
    except Exception as e:
        return {"error": str(e)}


print("Generating new evaluation template...")
EMPATHY_EVALUATION_PROMPT_TEMPLATE = generate_eval_template()
print("Template generated successfully!")
print(EMPATHY_EVALUATION_PROMPT_TEMPLATE)
```

## **Testing Our Initial Prompt**

Instrument the application to send traces to Phoenix:

```python
from openinference.instrumentation.openai import OpenAIInstrumentor

from phoenix.otel import register

tracer_provider = register(
    project_name="LLM-as-a-Judge", endpoint="https://app.phoenix.arize.com/v1/traces"
)
OpenAIInstrumentor().instrument(tracer_provider=tracer_provider)
```

Now that we have our baseline prompt, we need to set up two key components:

* **Task**: The LLM as a Judge evaluation, where the model scores chatbot responses based on empathy and emotional intelligence.
* **Evaluator**: A function that compares the LLM as a Judge output to the ground-truth labels from our dataset

Finally, we run our experiment. With this setup, we can measure how well our prompt initially performs.

```python
import pandas as pd

from phoenix.evals import (
    OpenAIModel,
    llm_classify,
)


def initial_prompt(input):
    response_classifications = llm_classify(
        dataframe=pd.DataFrame([{"AI_Response": input["AI_Response"]}]),
        template=EMPATHY_EVALUATION_PROMPT_TEMPLATE,
        model=OpenAIModel(model="gpt-4"),
        rails=list(map(str, range(1, 11))),
        provide_explanation=True,
    )
    score = response_classifications.iloc[0]["label"]
    return int(score)


def evaluate_response(input, output):
    expected_score = input["EI_Empathy_Score"]
    predicted_score = output
    return abs(expected_score - predicted_score) <= 2
```

```python
import nest_asyncio

from phoenix.experiments import run_experiment

nest_asyncio.apply()

initial_experiment = run_experiment(
    dataset, task=initial_prompt, evaluators=[evaluate_response], experiment_name="initial_prompt"
)
```

## **Iteration 1: Improve Accuracy**

If you find that your LLM as a Judge prompt has low accuracy, we can make adjustmenets to the prompt to improve that. In this section, we explore 2 techniques for this: few shot examples and keeping a human in the loop.

### Technique 1: Few Shot Examples

Few-shot examples help improve the accuracy of an LLM as a Judge prompt by providing clear reference points for evaluation. Instead of relying solely on general instructions, the model learns from labeled examples that demonstrate correct scoring and reasoning.

By including a mix of high, medium, and low-scoring responses, we help the model:

* Understand nuanced criteria like empathy and emotional intelligence.
* Reduce inconsistencies by aligning with real-world judgments.

```python
few_shot_examples = """
---
To help guide your reasoning, compare the text to the following examples and scorings:

Example 1: "You should try to focus and get your work done. Everyone has tough days."
  Score: 2
  Explanation: No acknowledgment of the user’s emotions, dismissive and offers no real support

Example 2: "That sounds really stressful. Have you considered talking to a supervisor or taking breaks?"
  Score: 5
  Explanation: Acknowledges stress, but in a generic way. Provides advice, but not very personal. Could be warmer in tone.

Example 3: "I’m really sorry you’re feeling this way. It’s completely understandable to feel overwhelmed. You’re not alone in this. Have you had a chance to take a break or talk to someone who can support you?"
  Score: 9
  Explanation: Validates emotions, reassures the user, and offers support
"""
EMPATHY_EVALUATION_PROMPT_TEMPLATE_IMPROVED = EMPATHY_EVALUATION_PROMPT_TEMPLATE + few_shot_examples
```

```python
def llm_as_a_judge(input):
    response_classifications = llm_classify(
        dataframe=pd.DataFrame([{"AI_Response": input["AI_Response"]}]),
        template=EMPATHY_EVALUATION_PROMPT_TEMPLATE_IMPROVED,
        model=OpenAIModel(model="gpt-4"),
        rails=list(map(str, range(1, 11))),
        provide_explanation=True,
    )
    score = response_classifications.iloc[0]["label"]
    return int(score)


experiment = run_experiment(
    dataset,
    task=llm_as_a_judge,
    evaluators=[evaluate_response],
    experiment_name="few_shot_examples",
)
```

### Technique 2: Human in the Loop

Keeping a human in the loop improves the accuracy of an LLM as a Judge by providing oversight, validation, and corrections where needed. In Phoenix, we can do this with [annotations](https://arize.com/docs/phoenix/tracing/features-tracing/how-to-annotate-traces). While LLMs can evaluate responses based on predefined criteria, human reviewers help:

* Catch edge cases and biases that the model may overlook.
* Refine scoring guidelines by identifying inconsistencies in LLM outputs.
* Continuously improve the prompt by analyzing where the model struggles and adjusting instructions accordingly.

However, human review can be costly and time-intensive, making full-scale annotation impractical. Fortunately, even a small number of human-labeled examples can significantly enhance accuracy.

![Human Annotation](https://storage.googleapis.com/arize-phoenix-assets/assets/images/human_annotation.gif)

## **Iteration 2: Reduce Bias**

### Style Invariant Evaluation

One common bias in LLM as a Judge evaluations is favoring certain writing styles over others. For example, the model might unintentionally rate formal, structured responses higher than casual or concise ones, even if both convey the same level of empathy or intelligence.

To reduce this bias, we focus on style-invariant evaluation, ensuring that the LLM judges responses based on content rather than phrasing or tone. This can be achieved by:

* Providing diverse few-shot examples that include different writing styles.
* Testing for bias by evaluating responses with varied phrasing and ensuring consistent scoring.

By making evaluations style-agnostic, we create a more robust scoring system that doesn’t unintentionally penalize certain tones.

```python
style_invariant = """
----
To help guide your reasoning, below is an example of how different response styles and tones can achieve similar scores:

#### Scenario: Customer Support Handling a Late Order
User: "My order is late, and I needed it for an important event. This is really frustrating."

Response A (Formal): "I sincerely apologize for the delay..."
Response B (Casual): "Oh no, that’s really frustrating!..."
Response C (Direct): "Sorry about that. I’ll check..."
"""
EMPATHY_EVALUATION_PROMPT_TEMPLATE_IMPROVED = EMPATHY_EVALUATION_PROMPT_TEMPLATE + style_invariant
```

```python
def llm_as_a_judge(input):
    response_classifications = llm_classify(
        dataframe=pd.DataFrame([{"AI_Response": input["AI_Response"]}]),
        template=EMPATHY_EVALUATION_PROMPT_TEMPLATE_IMPROVED,
        model=OpenAIModel(model="gpt-4"),
        rails=list(map(str, range(1, 11))),
        provide_explanation=True,
    )
    score = response_classifications.iloc[0]["label"]
    return int(score)


experiment = run_experiment(
    dataset, task=llm_as_a_judge, evaluators=[evaluate_response], experiment_name="style_invariant"
)
```

## **Iteration 3: Reduce Cost and Latency**

Longer prompts increase computation costs and response times, making evaluations slower and more expensive. To optimize efficiency, we focus on condensing the prompt while preserving clarity and effectiveness. This is done by:

* Removing redundant instructions and simplifying wording.
* Using bullet points or structured formats for concise guidance.
* Eliminating unnecessary explanations while keeping critical evaluation criteria intact.

A well-optimized prompt reduces token count, leading to faster, more cost-effective evaluations without sacrificing accuracy or reliability.

```python
def generate_condensed_template():
    meta_prompt = """
    You are an expert in prompt engineering and LLM evaluation. Your task is to optimize a given LLM-as-a-judge prompt by reducing its word count significantly while maintaining all essential information, including evaluation criteria, scoring system, and purpose.

    Requirements:
    Preserve all key details such as metrics, scoring guidelines, and judgment criteria.

    Eliminate redundant phrasing and unnecessary explanations.

    Ensure clarity and conciseness without losing meaning.

    Maintain the prompt’s effectiveness for consistent evaluations.

    Output Format:
    Return only the optimized prompt as plain text, with no explanations or commentary.

    """

    try:
        response = client.chat.completions.create(
            model="gpt-4",
            messages=[
                {
                    "role": "user",
                    "content": "Provided LLM-as-a-judge prompt"
                    + EMPATHY_EVALUATION_PROMPT_TEMPLATE,
                },
                {"role": "user", "content": meta_prompt},
            ],
            temperature=0.9,  # High temperature for more creativity
        )

        return response.choices[0].message.content
    except Exception as e:
        return {"error": str(e)}


print("Generating condensed evaluation template...")
EMPATHY_EVALUATION_PROMPT_TEMPLATE_CONDENSED = generate_condensed_template()
print("Template generated successfully!")
print(EMPATHY_EVALUATION_PROMPT_TEMPLATE_CONDENSED)
```

```python
def llm_as_a_judge(input):
    response_classifications = llm_classify(
        dataframe=pd.DataFrame([{"AI_Response": input["AI_Response"]}]),
        template=EMPATHY_EVALUATION_PROMPT_TEMPLATE_CONDENSED,
        model=OpenAIModel(model="gpt-4"),
        rails=list(map(str, range(1, 11))),
        provide_explanation=True,
    )
    score = response_classifications.iloc[0]["label"]
    return int(score)


experiment = run_experiment(
    dataset, task=llm_as_a_judge, evaluators=[evaluate_response], experiment_name="condensed_prompt"
)
```

## **Iteration 4: Self-Refinement (Iterative LLM as Judge)**

Self-refinement allows a Judge to improve its own evaluations by critically analyzing and adjusting its initial judgments. Instead of providing a static score, the model engages in an iterative process:

* Generate an initial score based on the evaluation criteria.
* Reflect on its reasoning, checking for inconsistencies or biases.
* Refine the score if needed, ensuring alignment with the evaluation guidelines.

By incorporating this style of reasoning, the model can justify its decisions and self-correct errors.

```python
refinement_text = """
---
After you have done the evaluation, follow these two steps:
1. Self-Critique
Review your initial score:
- Was it too harsh or lenient?
- Did it consider the full context?
- Would others agree with your score?
Explain any inconsistencies briefly.

2. Final Refinement
Based on your critique, adjust your score if necessary.
- Only output a number (1-10)
"""
EMPATHY_EVALUATION_PROMPT_TEMPLATE_IMPROVED = EMPATHY_EVALUATION_PROMPT_TEMPLATE + refinement_text
```

```python
def llm_as_a_judge(input):
    response_classifications = llm_classify(
        dataframe=pd.DataFrame([{"AI_Response": input["AI_Response"]}]),
        template=EMPATHY_EVALUATION_PROMPT_TEMPLATE_IMPROVED,
        model=OpenAIModel(model="gpt-4"),
        rails=list(map(str, range(1, 11))),
        provide_explanation=True,
    )
    score = response_classifications.iloc[0]["label"]
    return int(score)


experiment = run_experiment(
    dataset, task=llm_as_a_judge, evaluators=[evaluate_response], experiment_name="self_refinement"
)
```

## **Iteration 5: Combining Techniques**

To maximize the accuracy and fairness of our Judge, we will combine multiple optimization techniques. In this example, we will incorporate few-shot examples and style-invariant evaluation to ensure the model focuses on content rather than phrasing or tone.

By applying these techniques together, we aim to create a more reliable evaluation framework.

```python
EMPATHY_EVALUATION_PROMPT_TEMPLATE_IMPROVED = (
    EMPATHY_EVALUATION_PROMPT_TEMPLATE + few_shot_examples + style_invariant
)
```

```python
def llm_as_a_judge(input):
    response_classifications = llm_classify(
        dataframe=pd.DataFrame([{"AI_Response": input["AI_Response"]}]),
        template=EMPATHY_EVALUATION_PROMPT_TEMPLATE_IMPROVED,
        model=OpenAIModel(model="gpt-4"),
        rails=list(map(str, range(1, 11))),
        provide_explanation=True,
    )
    score = response_classifications.iloc[0]["label"]
    return int(score)


experiment = run_experiment(
    dataset, task=llm_as_a_judge, evaluators=[evaluate_response], experiment_name="combined"
)
```

## **Final Results**

Techniques like few-shot examples, self-refinement, style-invariant evaluation, and prompt condensation each offer unique benefits, but their effectiveness will vary depending on the task.

{% hint style="info" %}
**Note**: You may sometimes see a decline in performance, which is not necessarily "wrong." Results can vary due to factors such as the choice of LLM and other inherent model behaviors.
{% endhint %}

By systematically testing and combining these approaches, you can refine your evaluation framework.

![Final Results](https://storage.googleapis.com/arize-phoenix-assets/assets/images/llm_as_a_judge_tutorial_results.png)
