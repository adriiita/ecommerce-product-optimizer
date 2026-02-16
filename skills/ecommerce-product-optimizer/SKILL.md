---
name: ecommerce-product-optimizer
description: >
  This skill should be used when the user asks to "optimize product listings",
  "process a Shopify CSV", "generate SEO content for products", "bulk optimize
  products", "enrich my product CSV", or provides a CSV file with product data
  and wants SEO-optimized ecommerce content generated for each row. Also triggers
  when the user mentions "product descriptions", "Shopify listings", "SEO titles
  for products", or "FAQ generation for products".
version: 0.1.0
---

# Ecommerce Product Listing Optimizer

## Purpose
Process CSV files containing ecommerce product data and generate SEO-optimized listings using **parallel subagents**. Each product row is processed by an independent subagent to prevent cross-product contamination and keep context windows clean. Default output format is Shopify-compatible, but the approach works for any ecommerce platform.

## When to Use
User provides a CSV with product data and wants:
- SEO-optimized titles, descriptions, and body content
- FAQ generation per product
- Shopify-ready structured data (Handle, Tags, HTML Body, etc.)
- Bulk processing of 5–100+ products efficiently

## Processing Modes

The user controls how processing runs. Detect the mode from their instructions:

| User Says | Mode | Behavior |
|-----------|------|----------|
| (default / no mention) | **Parallel batched** | 5–8 subagents per batch, all batches run sequentially |
| "just do one", "test one row", "single row" | **Single test** | Process exactly 1 row (first row or user-specified), show full result for review |
| "one at a time", "sequential", "no parallel" | **Sequential** | 1 subagent at a time, show result after each, wait for user OK to continue |
| "process row 5", "only the tennis racket" | **Targeted** | Process specific row(s) identified by number or product name match |
| "all parallel", "max speed", "run them all" | **Full parallel** | All rows in one batch (use only if row count ≤ 8, otherwise fall back to batched) |

**Key rule**: If the user says to use a single agent for a single task, respect that exactly. Do NOT parallelize unless they ask for it or accept it.

---

## User Control & Interaction

**Default behavior: Just run it.** Don't add unnecessary pauses or confirmations. Parse the CSV, process all rows, deliver the result.

The user is in control. Respond to what they ask for — don't impose checkpoints they didn't request.

### When to pause and ask the user:
- **Ambiguous column mapping**: If you genuinely can't figure out which column is which, ask. If it's obvious, just map it and go.
- **User explicitly asks to review**: "show me first", "let me check before you continue", "do one row first so I can see" → then pause and show.
- **Sequential mode**: If the user asked for one-at-a-time processing, show each result and wait for their go-ahead before the next.
- **Errors**: If multiple rows fail, pause and let the user know before continuing.

### When NOT to pause:
- Column mapping is clear → just proceed
- User said "optimize this CSV" with no caveats → run everything, report at the end
- User already validated prompt/quality in a previous run → don't re-ask

### Always provide at the end:
- Quick summary: "X of Y products optimized, saved to [file]"
- List any failures by product name
- Offer to show specific results only if there were issues

---

## Workflow Overview

```
CSV Input → Parse & Validate → Determine Mode → Process (parallel/sequential/single) → Merge Results → Save Enriched CSV → Report
```

---

## Phase 1: Parse & Validate the CSV

### Step 1: Read the CSV
Use Python to read the CSV and inspect its structure:

```python
import pandas as pd

df = pd.read_csv('/path/to/input.csv')
print(f"Total rows: {len(df)}")
print(f"Columns: {list(df.columns)}")
print(df.head(2).to_string())
```

### Step 2: Map Columns
The input CSV may use different column names. Map them to these standard fields:

| Standard Field   | Possible CSV Column Names                              | Required? |
|------------------|-------------------------------------------------------|-----------|
| `product_name`   | Product Name, Title, Name, Product Title               | YES       |
| `brand`          | Brand, Vendor                                          | YES       |
| `category`       | Category, Type, Product Type, Product Category         | YES       |
| `description`    | Description, Body, Content, Product Description        | YES       |
| `price`          | Price, Variant Price, VariantPrice                     | YES       |
| `sku`            | SKU, Variant SKU, VariantSKU, Product SKU              | YES       |
| `keywords`       | Keywords, Tags, SEO Keywords, Target Keywords          | Optional  |
| `language`       | Language, Lang (default: "English")                    | Optional  |
| `notes`          | Notes, Additional Notes                                | Optional  |
| `benefits`       | Beneficios, Benefits, Key Benefits                     | Optional  |

If a column mapping is ambiguous, ask the user to confirm.

### Step 3: Determine Row Selection
- **Default**: Process ALL rows
- **If user specifies a filter**: Apply it before processing
  - Row range: "rows 5-10" → `df.iloc[4:10]`
  - Column filter: "only Spanish products" → `df[df['Language'] == 'Spanish']`
  - Value filter: "only products over $100" → `df[df['Price'] > 100]`
- Report to user: "Processing X of Y total rows"

---

## Phase 2: Process Products

### Mode-Specific Execution

**Parallel Batched (default)**:
- Batch size: 5–8 subagents per round
- For 15 rows → 2 batches (8 + 7); for 30 rows → 4 batches (8 + 8 + 8 + 6)
- Launch ALL subagents in a batch using **multiple Task tool calls in a SINGLE message**
- Wait for all in a batch to complete before starting the next

**Single Test**:
- Process exactly 1 row (first row or user-specified row)
- Use a single Task tool call
- Show the full result to the user

**Sequential**:
- 1 Task tool call at a time
- Show result after each
- Wait for user to say continue / stop / adjust before next row

**Targeted**:
- Process only the specific row(s) the user identified
- Can be parallel (if multiple targeted rows) or single

### Task Tool Configuration
For each row, make a Task tool call with:
- **`subagent_type`**: `"general-purpose"`
- **`description`**: `"Optimize: [first 3 words of product name]"`
- **`prompt`**: The full subagent prompt (see below) with all placeholders replaced by actual row data

### Constructing the Subagent Prompt
For EACH row, replace every `{placeholder}` with the actual value from that CSV row. If a field is empty/missing, use "Not provided".

---

## SUBAGENT PROMPT TEMPLATE

```
You are an expert ecommerce product listing optimizer. Your job is to create ONE SEO-optimized Shopify product listing.

CRITICAL: You must describe EXACTLY the product below. Do NOT substitute a different product. Do NOT use examples from training data.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PRODUCT DATA:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Product Name: {product_name}
Brand: {brand}
Category: {category}
Current Description: {description}
Price: {price}
SKU: {sku}
Target Keywords: {keywords}
Key Benefits: {benefits}
Additional Notes: {notes}
Required Language: {language}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

STEP 1 — IDENTIFY CATEGORY TYPE
Based on "{category}", determine which category this product belongs to:
- FASHION & APPAREL → Write about: fabric, fit, style, color, occasion, design details
- JEWELRY & ACCESSORIES (including Watches) → Write about: materials, craftsmanship, size, style, occasion
- ELECTRONICS & TECH → Write about: specs, battery, connectivity, compatibility, performance
- BEAUTY & COSMETICS → Write about: ingredients, skin benefits, application, texture
- HOME & DECOR → Write about: materials, dimensions, style, functionality
- SPORTS & FITNESS → Write about: performance, durability, activity type, features
- FOOD & BEVERAGE → Write about: taste, ingredients, nutrition, origin
- BABY & KIDS → Write about: safety, age range, development, materials

NEVER cross-contaminate categories. Do not mention battery life for clothing. Do not mention fabric for electronics.

STEP 2 — GENERATE CONTENT
All content MUST be in {language}.

1. Handle: URL-friendly slug of the product name (lowercase, hyphens, no special chars)

2. Title: Compelling product title in {language}

3. SEO Title: EXACTLY 50–60 characters. Count carefully.
   - Include the primary keyword and brand
   - Example counting: "P(1)r(2)e(3)m(4)i(5)u(6)m(7)" = 7 chars

4. SEO Description: EXACTLY 150–160 characters. Count carefully.
   - Include: main keyword + key benefit + call-to-action

5. Body (HTML): Category-appropriate content with:
   <h2>[Key benefit headline]</h2>
   <p>[2-3 sentences about THIS product's appeal and use case]</p>
   <h3>[Features heading]</h3>
   <ul>
     <li><strong>[Feature 1]:</strong> [Detail from provided description]</li>
     <li><strong>[Feature 2]:</strong> [Detail from provided description]</li>
     <li><strong>[Feature 3]:</strong> [Detail from provided description]</li>
     <li><strong>[Feature 4]:</strong> [Detail from provided description]</li>
   </ul>
   <h3>[Product Details heading]</h3>
   <ul>
     <li><strong>[Brand label]:</strong> {brand}</li>
     <li><strong>[Category label]:</strong> {category}</li>
     <li><strong>[SKU label]:</strong> {sku}</li>
   </ul>
   <h3>[Care/Usage heading]</h3>
   <p>[Care or usage guidance appropriate to the product category]</p>

6. Tags: 12–15 comma-separated tags including:
   - All provided keywords
   - Brand name
   - Product type
   - Color/style descriptors from description
   - Use-case tags
   - All tags in {language}

7. 3 FAQ pairs:
   - Specific to THIS product and category
   - Answer real customer concerns
   - In {language}

STEP 3 — VALIDATE
Before outputting, verify:
☐ Title describes "{product_name}" — not a different product
☐ Body HTML is about "{product_name}" — not a different product
☐ No features from wrong category mentioned
☐ Brand is exactly: {brand}
☐ SKU is exactly: {sku}
☐ Price is exactly: {price}
☐ ALL text is in {language}
☐ SEO Title is 50-60 characters
☐ SEO Description is 150-160 characters
☐ FAQs are relevant to THIS product

OUTPUT — Return ONLY valid JSON. No markdown code blocks. No explanation. Just the JSON object:

{"Handle": "url-slug-here", "Title": "Product title here", "Body": "<h2>...</h2><p>...</p>...", "Vendor": "{brand}", "Type": "{category}", "Tags": "tag1, tag2, tag3, ...", "Published": true, "Variant_SKU": "{sku}", "Variant_Price": {price}, "SEO_Title": "50-60 char title", "SEO_Description": "150-160 char description", "FAQ1_Question": "Question?", "FAQ1_Answer": "Answer", "FAQ2_Question": "Question?", "FAQ2_Answer": "Answer", "FAQ3_Question": "Question?", "FAQ3_Answer": "Answer"}
```

---

## Phase 3: Collect & Merge Results

### Parsing Subagent Responses
Each subagent returns a text response. Extract the JSON from it:

```python
import json
import re

def parse_subagent_result(result_text):
    """Extract JSON from subagent response, handling common issues."""
    # Try direct parse first
    try:
        return json.loads(result_text.strip())
    except json.JSONDecodeError:
        pass

    # Try to find JSON object in the text
    match = re.search(r'\{[\s\S]*\}', result_text)
    if match:
        try:
            return json.loads(match.group())
        except json.JSONDecodeError:
            pass

    return None  # Parsing failed
```

### Error Handling
- If JSON parsing fails: **retry that row once** with the same prompt, adding "IMPORTANT: Return ONLY valid JSON, no other text." to the end
- If retry also fails: add the row with empty generated columns and set an `Optimization_Error` column to "Failed to generate"
- Always report: "✓ X of Y products optimized successfully" (and list any failures)

### Merging with Original CSV
```python
import pandas as pd

# original_df = the parsed input CSV (only rows that were processed)
# results = list of parsed JSON dicts, one per processed row, in order

# Define expected output columns
output_columns = [
    'Handle', 'Title', 'Body', 'Vendor', 'Type', 'Tags', 'Published',
    'Variant_SKU', 'Variant_Price', 'SEO_Title', 'SEO_Description',
    'FAQ1_Question', 'FAQ1_Answer', 'FAQ2_Question', 'FAQ2_Answer',
    'FAQ3_Question', 'FAQ3_Answer'
]

# Build results DataFrame
results_df = pd.DataFrame(results)

# Ensure all expected columns exist (fill missing with empty string)
for col in output_columns:
    if col not in results_df.columns:
        results_df[col] = ''

# Prefix generated columns to avoid name collisions with original CSV
# Only prefix if there's an actual collision
original_cols = set(original_df.columns)
for col in results_df.columns:
    if col in original_cols:
        results_df = results_df.rename(columns={col: f'Generated_{col}'})

# Merge: original columns + generated columns side by side
enriched_df = pd.concat(
    [original_df.reset_index(drop=True), results_df.reset_index(drop=True)],
    axis=1
)

# Save
output_filename = input_filename.replace('.csv', '_optimized.csv')
enriched_df.to_csv(output_path, index=False)
```

---

## Phase 4: Output & Reporting

### Output Location
- **If the user has a workspace folder mounted**: Save the enriched CSV there (this is the default)
- **If the user specifies an output folder**: Save there instead (e.g., "save it in my exports folder")
- **File naming**: `[original_filename]_optimized.csv` (e.g., `Luxury Watches Collection_optimized.csv`)
- Always provide a clickable `computer://` link to the saved file so the user can open it directly

### Summary
1. Save the enriched CSV to the output location
2. Report to user:
   - Total rows processed
   - Successful optimizations
   - Any failures (with product names)
   - Output file path with link
3. Offer to show specific results only if there were issues

---

## Configuration Notes

### Model Selection for Subagents
- Default: inherit parent model (recommended for quality)
- For cost optimization on large batches (50+ rows): consider using `model: "haiku"` in the Task tool, but note that quality may decrease for complex categories
- For highest quality: use `model: "sonnet"` or `model: "opus"`

### Customizing the Prompt
The subagent prompt above is optimized for Shopify but works for any ecommerce platform. To customize:
- Modify the Body HTML template structure for different platforms
- Adjust the output JSON fields to match your target platform's schema
- Change tag count or FAQ count as needed

### Scaling Guidance
| Row Count | Batch Size | Expected Batches | Notes                          |
|-----------|-----------|------------------|--------------------------------|
| 1–8       | All at once | 1              | Single batch                   |
| 9–20      | 8         | 2–3              | Standard approach              |
| 21–50     | 8         | 3–7              | Monitor for rate limits        |
| 50–100    | 5         | 10–20            | Reduce batch size for safety   |
| 100+      | 5         | 20+              | Consider chunking CSV first    |

---

## Quick Reference — The Full Flow

```
1. User provides CSV + optional instructions (filtering, mode, language, etc.)
2. YOU read this SKILL.md
3. Parse CSV with Python (pandas) via Bash tool
4. Map columns (ask user ONLY if genuinely ambiguous)
5. Detect processing mode from user instructions (default: parallel batched)
6. Process:
   - PARALLEL: For each batch of 5-8 rows, launch N Task calls in ONE message
   - SINGLE: Launch 1 Task call, show result
   - SEQUENTIAL: Launch 1 Task call, show result, wait for user, repeat
   - TARGETED: Process only specified rows (parallel or single depending on count)
7. Parse JSON from each subagent result
8. Merge all results with original CSV using Python
9. Save enriched CSV to output folder
10. Report summary: "X/Y optimized, saved to [file]" + any failures
```
