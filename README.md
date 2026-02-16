# Ecommerce Product Optimizer

Bulk-optimize ecommerce product listings from CSV files using parallel subagents. Generates SEO-optimized titles, descriptions, HTML body content, FAQs, and tags — Shopify-ready by default, works for any ecommerce platform.

## What it does

Drop a CSV with product data (name, brand, category, description, price, SKU) and get back the same CSV enriched with 17 new columns: optimized titles, SEO metadata, HTML product descriptions, tags, and 3 FAQ pairs per product.

Each product row is processed by an independent subagent, so there's no cross-contamination between products. Processing runs in parallel batches for speed.

## Components

| Component | Name | Purpose |
|-----------|------|---------|
| Skill | `ecommerce-product-optimizer` | Core optimization logic, subagent prompt, merge workflow |
| Command | `/optimize-products` | Quick-launch the optimizer from any conversation |

## Usage

### Slash command
```
/optimize-products path/to/products.csv
```

Or just drop a CSV into the conversation and say "optimize my products."

### Processing modes

- **Default**: Parallel batched (5-8 products at a time)
- **"just do one"**: Single row test
- **"one at a time"**: Sequential with approval between rows
- **"only row 5"**: Targeted specific rows
- **"only products over $100"**: Filtered by value

### Input CSV format

Required columns: Product Name, Brand, Category, Description, Price, SKU

Optional columns: Keywords, Language, Notes, Benefits/Beneficios

Column names are flexible — the skill auto-maps common variations.

### Output

Original CSV + 17 new columns appended: Generated_Handle, Title, Body (HTML), Vendor, Type, Tags, Published, Variant_SKU, Variant_Price, SEO_Title, SEO_Description, FAQ1-3 Question/Answer pairs.

## Setup

No external services or API keys required. The plugin uses Claude's built-in Task tool for parallel subagent processing.
