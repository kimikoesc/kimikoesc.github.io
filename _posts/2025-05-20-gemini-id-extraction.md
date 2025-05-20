---
title: "Extracting Government ID Information with Gemini and Structured Output"
date: 2025-05-20 00:00:00 +0000
categories: [AI]
tags: ["ai", "gemini", "ocr", "golang"]
draft: false
---

Our app requires users to submit personal details such as their name and government-issued ID number (e.g., an IC card or passport). Initially, we relied on manual input: a simple text form that users filled out themselves.

Unsurprisingly, this approach led to significant issues. Users frequently entered typos or provided incomplete information. In some cases, they abandoned the app entirely due to the friction. To improve the onboarding experience and ensure data accuracy, we introduced a new feature: photo-based ID submission.

---

## Why We Use Gemini Instead of 3rd Party OCR Services

There are plenty of commercial ID-scanning APIs on the market that offer robust OCR pipelines with validation layers and fraud detection. However, most of them are paid services, and costs can ramp up quickly, especially at scale.

We decided to build our own ID extraction pipeline using [Gemini](https://ai.google.dev/gemini-api/docs) instead. It offered us more control, flexibility, and most importantly, cost-efficiency. Our infrastructure is optimized for latency and cost, and we found Gemini's image understanding capabilities were strong enough to justify building a lightweight in-house solution.

---

## ID Extraction Using Gemini + Structured Output

We send the uploaded image of the ID to Gemini, along with a short instruction prompt. Rather than parsing the response ourselves from raw text, we use **structured output** with a predefined JSON schema. Here's why this matters:

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
			"government_issued_id": {...},
			"government_issued_name": {...},
			"is_valid_id": {...},
			"country": {...},
			"id_type": {...},
			"confidence_level": {...},
			"card_condition": {...},
		},
		Required: []string{
			"government_issued_id",
			"government_issued_name",
			"is_valid_id",
			"country",
			"id_type",
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
The image is passed via `InlineData`, and Gemini returns a clean JSON response matching our schema.

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
Gemini’s processing time is acceptable for our flow, and using structured outputs helped reduce token count, making the cost manageable.

- **Fallback Flow**
We built a simple retry-and-review logic around these outputs. If the image fails validation, users can try again or request manual help.

---

## Final Thoughts

This feature started as a cost-saving experiment, but the results exceeded expectations. Gemini gave us a flexible and affordable way to parse structured data from ID images without relying on expensive 3rd party APIs.

For teams considering OCR pipelines for onboarding or identity verification, Gemini’s structured output feature combined with its image understanding API is a strong alternative — especially if you want full control over the data flow and pricing.