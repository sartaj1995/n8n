WIP

# MENA Tech & Vision 2030 Intelligence Tracker

## Business Problem Statement

Management consultants, market analysts, and investment strategy teams working within the Middle East and North Africa (MENA) region must keep up with a relentless firehose of news to track fast-moving macroeconomic shifts. Manually parsing hundreds of news feeds daily to isolate critical updates regarding Vision 2030, giga-projects like NEOM, and state-backed investments is time-consuming and leads to high-signal updates getting lost in general media noise.

This workflow transforms passive news monitoring into an active market intelligence pipeline. By combining robust web scraping failovers, custom keyword-filtering automation, and advanced LLM synthesis, it aggregates critical market changes, packages them into a clean executive format, and drops a consolidated daily briefing directly into a consultant’s inbox.

## System Architecture & Data Flow

The automation utilizes an advanced, resilient multi-route design built in n8n. It features an integrated failover system that handles unexpected source network blocks automatically.Primary Trigger (RSS Read Node): Scheduled to run daily. It attempts to pull and parse the targeted regional news feed directly into structured data.Failover Backup Path (If Node & HTTP Request): If the primary RSS node encounters a network restriction or security block (such as a 403 Forbidden error), an integrated error-handling parameter passes execution to a fallback branch. An HTTP Request Node with a spoofed browser User-Agent header overrides the block, scrapes the backup source data, and routes it through an HTML Extract Node to maintain workflow continuity.Triage & Extraction Layer (Code in Python): A unified Native Python node processes data arriving from either the primary or failover path. It acts as an automated filter, running item-by-item text matching across the titles and article summaries against specific high-value keywords like NEOM, Aramco, giga-projects, ai investment, and Vision 2030.Synthesis Layer (Advanced AI Chain): Filtered articles are fed into a Basic LLM Chain connected to an OpenAI or Anthropic chat model. The AI functions as a market analyst, distilling long-form articles into high-impact, executive summaries mapping back to strategic regional initiatives.Consolidation & Delivery (Python Combine + Gmail API): To prevent inbox flooding, a final Python script loops through all generated summaries from that day, compiles them into a single HTML digest layout, and pushes the finalized brief to the user's inbox via the Gmail node.

## Technical Specifications & Field Mapping

n8n StepApp Node ComponentExecution/Logic EventCore Data Mapping & ConfigurationStep 1RSS ReadRead FeedPulls daily updates. Settings configured to On Error: Continue to feed the failover logic.Step 2If NodeError ValidationChecks if {{ $json.error }} Exists. True routes to fallback path; False routes directly to filtering.Step 3HTTP RequestGET WebpageTriggers on True. Header injected: User-Agent: Mozilla/5.0... to bypass 403 blocks.Step 4HTML ExtractExtract ElementsTarget source data: data text block. Isolates individual stories into clean JSON objects.Step 5Code (Python)Run Once for All ItemsLoops through _items. Converts strings to lowercase, evaluating matches for neom, aramco, etc.Step 6Basic LLM ChainPredictUtilizes gpt-4o-mini or claude-3-5-haiku. Maps {{ $json.title }} and {{ $json.content }} to the prompt.Step 7Code (Python)Compile ListLoops through compiled items to build a single HTML variable string: {{ $json.email_body }}.Step 8GmailSend MessageBody Type: HTML. Body Expression: {{ $json.email_body }}.

## Code Snippets

1. Keyword Filtering Script (Python Filter Node)This Native Python script runs once across all items, keeping only data payloads that match targeted strategic terms.

```
Python
# Set up target keywords (all lowercase for case-insensitive matching)
keywords = ["neom", "aramco", "giga-projects", "ai investment", "vision 2030"]

filtered_items = []

# In Native Python, incoming data is instantly accessible via '_items'
for item in _items:
    data = item.get("json", {})
    
    # Grab text fields safely
    title = str(data.get("title", "")).lower()
    content = str(data.get("content", data.get("snippet", data.get("text", "")))).lower()
    
    # Check if keywords appear in either text block
    if any(keyword in title or keyword in content for keyword in keywords):
        filtered_items.append({"json": data})

return filtered_items
```

2. Digest Consolidation Script (Python Combine Node)This snippet aggregates multiple items down into a single item containing an HTML string, preventing n8n from running subsequent email nodes in a loop.

```
Python
full_digest = "<h2>Daily MENA Tech & Vision 2030 Intelligence Tracker</h2><br>"

for item in _items:
    summary = item.get("json", {}).get("text", "")
    
    if summary:
        full_digest += f"<div style='margin-bottom: 20px;'>{summary}</div><hr>"

return [{"json": {"email_body": full_digest}}]
```

## Implementation & Customization

1. Customizing Tracker KeywordsTo track alternative regional priorities (e.g., specific country names like uae, oman, or target sectors like fintech, hydrogen), open the Python Filter Node and update the strings inside the keywords array:

```
Python
keywords = ["your_keyword_1", "your_keyword_2", "your_keyword_3"]
```

2. LLM Prompt TuningTo alter the tone or output formatting of your summary, adjust the prompt text inside your Basic LLM Chain Node:
Switch the Prompt parameter to Define Below.
Ensure that you instruct the LLM to write using raw HTML tags (e.g., <strong>, <p>) rather than standard markdown formatting flags like  or ###, preventing styling syntax errors in the final email view.
