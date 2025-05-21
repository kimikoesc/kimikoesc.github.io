---
title: "Extracting Government ID Information with Gemini and Structured Output"
date: 2025-05-20 00:00:00 +0000
categories: [AI]
tags: ["ai", "gemini", "ocr", "golang"]
draft: false
---

How do you KYC?  Many fintechs need a Know Your Customer step for due diligence, there are many 3rd party services available to perform this but in the age of AI, I was interested to see if they’re still needed.

---

## Why I Use AI Instead of 3rd Party OCR Services

There are plenty of commercial ID-scanning APIs on the market that offer robust OCR pipelines with validation layers and fraud detection. However, most of them are paid services, and costs can ramp up quickly, especially at scale.

I decided to build my own ID extraction pipeline using AI instead. It offered more control, flexibility, and most importantly, cost-efficiency.

---

## Why Gemini

Gemini is a great fit for ID extraction thanks to its combination of vision and language capabilities, structured output, and low overhead.

- **Image + Language Understanding** – Gemini can [analyze ID images](https://ai.google.dev/gemini-api/docs/image-understanding) and extract structured information in one step, without needing a separate OCR layer.
- **Native JSON Output** – With structured output, we skip brittle text parsing and get clean, ready-to-use data directly from the model.
- **Simple and Scalable** – No vendor lock-in or per-scan fees — just a flexible, prompt-based approach that scales with usage.

---

## ID Extraction Using Gemini + Structured Output

We send the uploaded image of the ID to Gemini, along with a short instruction prompt. Rather than parsing the response from raw text, we use **structured output** with a predefined JSON schema. Here's why this matters:

- **Reduced token usage** – Using `ResponseSchema` saves tokens compared to crafting long, example-heavy prompts.
- **Cleaner integration** – We get ready-to-parse data without relying on brittle string parsing.
- **Validation by design** – We enforce required fields and types upfront.

### Example (Go):

```go
config := &genai.GenerateContentConfig{
	ResponseMIMEType: "application/json",
	ResponseSchema: &genai.Schema{
		Type: genai.TypeObject,
		Properties: map[string]*genai.Schema{
			"gov_id_number": {...},
			"gov_id_name": {...},
			"is_valid_id": {...},
			"country": {...},
			"gov_id_type": {...},
			"confidence_level": {...},
			"card_condition": {...},
		},
		Required: []string{
			"gov_id_number",
			"gov_id_name",
			"is_valid_id",
			"country",
			"gov_id_type",
			"confidence_level",
			"card_condition",
		},
	},
}
```

The actual prompt sent is minimal:

```go
Text: "Analyze the provided image of an ID card. Extract the government issued number and the full name."
```
The image is passed via `InlineData`, and Gemini returns a clean JSON response matching the schema.

---

## The Fields That Matter Most

Since we have no control over the images users upload, we needed ways to validate the extraction quality. Some users submit selfies, photos of unrelated documents, or IDs that are too blurry to read.

To help with this, we added these fields to the schema:

- `is_valid_id`: Whether the image contains a recognizable ID card.
- `confidence_level`: A percentage score indicating how sure the model is about the extraction.
- `card_condition`: A general assessment of the card’s readability (e.g., GOOD, OK, BAD).

We use these values to determine the next step:

- If confidence is high and the card is valid → we auto-fill the user’s data.
- If confidence is low → we prompt the user to retake the photo or escalate to human review.

---

## Challenges and Observations

- **Garbage In, Garbage Out**

Gemini performs well, but only if the image is clear. Low-quality images lead to bad results. `is_valid_id` and `confidence_level` help catch this early.

- **Wide Format Variation**

IDs vary across countries and card types. The structured schema helps generalize across formats without needing to re-prompt or post-process too heavily.

- **Latency and Cost Balance**

Gemini’s processing time is acceptable for the flow, and using structured outputs helped reduce token count, making the cost manageable.

- **Fallback Flow**

We built a simple retry-and-review logic around these outputs. If the image fails validation, users can try again or request manual help.

---

## Final Thoughts

This feature started as a cost-saving experiment, but the results exceeded expectations. Gemini gave us a flexible and affordable way to parse structured data from ID images without relying on expensive 3rd party APIs.

For teams considering OCR pipelines for onboarding or identity verification, Gemini’s structured output feature combined with its image understanding API is a strong alternative — especially if you want full control over the data flow and pricing.