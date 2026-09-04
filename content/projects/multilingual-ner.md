---
title: "Multilingual Social-Media Named Entity Recognition"
description: "Transformer-based NER for noisy Turkish social-media text, with annotation redesign, model benchmarking, reproducible training and measurable evaluation gains."
slug: "multilingual-ner"
draft: false
---

[← Back to projects](/#projects)

**Context:** Machine Learning Engineer Intern · EnlightyAI · May–October 2025  
**Focus:** Named Entity Recognition · Transformer fine-tuning · Model evaluation

I worked on a transformer-based Named Entity Recognition system for noisy social-media text, with a strong focus on **data quality, reproducible experimentation, model comparison and span-level evaluation**.

The project was not only a model-training task. Improving performance required coordinating annotation, refining the entity schema, cleaning the training data, benchmarking several transformer architectures and analysing where the system failed.

## Project snapshot

| Metric | Result |
| --- | --- |
| Span-level micro-F1 | **0.44 → 0.75** |
| Relative F1 improvement | **~70%** |
| Annotated and validated data | **6,000 tweets** |
| Language split | **5,000 Turkish · 1,000 English** |
| Models benchmarked | **5 BERT / XLM-R variants** |
| Final entity schema | **9 entity types** |
| Annotation team | **8 people** |

## The challenge

Named Entity Recognition on social-media text is substantially harder than on clean, formal text. Tweets contain spelling variation, abbreviations, inconsistent punctuation, informal language, brand and product references and ambiguous entity boundaries.

The initial system reached a **0.44 span-level micro-F1**, leaving significant room for improvement. Rather than treating the model architecture as the only source of error, I worked across the full ML workflow:

1. annotation quality,
2. preprocessing,
3. entity-schema design,
4. model selection,
5. fine-tuning,
6. validation,
7. error analysis.

## Data and annotation

I coordinated an **8-person team** working on the annotation and validation of **6,000 tweets**, including **5,000 Turkish** and **1,000 English** samples.

A major part of the work was improving annotation consistency. I helped develop structured annotation guidelines and quality checks so that the training data better represented difficult social-media cases.

The annotation schema was expanded from earlier **3- and 7-entity configurations to 9 entity types**, increasing coverage for concepts such as brands, products and social-media-specific language.

This was important because inconsistent or incomplete labels directly limited what the model could learn.

## Model development

I benchmarked **five transformer-based NER models** from the BERT and XLM-R families using **PyTorch, TensorFlow and Hugging Face Transformers**.

The workflow included:

- preprocessing and dataset preparation,
- token-label alignment,
- transformer fine-tuning,
- validation,
- model comparison,
- prediction inspection,
- annotation-error analysis,
- failure analysis.

Among the evaluated architectures, **BERTurk-SUNLP** produced the strongest result for the target task.

## Results

The final system improved **span-level micro-F1 from 0.44 to 0.75**.

That represents:

- **+0.31 absolute F1**
- **~70% relative improvement**

The gain came from improving the system as a whole rather than relying on model replacement alone. Better annotation quality, cleaner preprocessing, model benchmarking and systematic error analysis all contributed to the final result.

## Reproducible ML workflow

To make experiments comparable, I built a reproducible workflow on **AWS EC2** covering:

```text
Raw annotated text
        ↓
Preprocessing and validation
        ↓
Token / entity alignment
        ↓
Transformer fine-tuning
        ↓
Model validation
        ↓
Span-level evaluation
        ↓
Prediction and error analysis
```

The workflow used **Hugging Face, PyTorch, AWS EC2 and Git** to keep model experiments, training changes and evaluation results consistent across multiple candidate architectures.

## What I contributed

- Fine-tuned and evaluated transformer-based NER models.
- Improved annotation and preprocessing workflows.
- Benchmarked five BERT/XLM-R model variants.
- Coordinated annotation and validation across an eight-person team.
- Helped expand the entity schema to nine entity types.
- Analysed prediction errors, annotation problems and data-quality issues.
- Built a reproducible AWS-based training and evaluation workflow.

## What this project demonstrates

This project is representative of how I approach applied machine learning: **measure the system, inspect failure modes, improve the data as well as the model and keep experiments reproducible**.

**Core technologies:** Python · PyTorch · Hugging Face Transformers · BERT · XLM-R · AWS EC2 · Git · NLP · Named Entity Recognition · Model Evaluation

[← Back to projects](/#projects)
