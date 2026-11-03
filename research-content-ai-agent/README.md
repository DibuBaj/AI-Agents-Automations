# Research to Instagram and TikTok Content (Chat)

An n8n chat workflow that turns a topic into research-backed Instagram and TikTok
draft content. It searches the web, extracts readable page text, keeps only
findings supported by multiple independent sources, and then writes platform-ready
copy using a configurable brand voice.

> **Important:** This workflow generates drafts only. It does not publish to
> Instagram, TikTok, or any other social platform.

## What the workflow does

1. Receives a topic through the n8n Chat Trigger.
2. Normalizes the chat input and rejects topics shorter than five characters.
3. Searches Tavily using advanced search and requests up to 15 candidate results.
4. Sorts candidates by Tavily score, removes duplicate domains, and keeps up to
   eight sources scoring at least `0.5`.
5. Splits the sources and retrieves their content through Jina Reader as
   Markdown/plain text.
6. Cleans the extracted content, removes very short pages and duplicate page
   starts, and limits each source to 10,000 characters.
7. Sends the collected sources to an Ollama-hosted `gemma4:31b` model. The analyst
   may return a finding only when at least three different sources explicitly
   support it.
8. Verifies the analyst JSON, source IDs, and minimum-source rule in an n8n Code
   node. The verified research briefing includes source links and support counts.
9. Sends the briefing, company, audience, and brand voice to a second Ollama
   `gemma4:31b` model to draft Instagram and TikTok content.
10. Validates the draft JSON and checks character, word, hashtag, and number
    traceability limits before returning the final response in chat.

## Included file

| File | Description |
| --- | --- |
| `Research to Instagram and TikTok Content (Chat).json` | n8n workflow export |

## Requirements

- An n8n instance with the LangChain nodes available.
- Internet access from n8n to:
  - `api.tavily.com`
  - `r.jina.ai`
  - the configured Ollama API endpoint
- A Tavily account and API key.
- A Jina account/API key if your Jina Reader setup requires authentication.
- An Ollama account/API endpoint that provides the `gemma4:31b` model.

The workflow uses these n8n node families:

- Core Code, Set, If, HTTP Request, Split Out, and Chat Trigger nodes.
- `@n8n/n8n-nodes-langchain` Chain LLM and Ollama Chat Model nodes.

## Required credentials

Create these credentials in n8n before importing or activating the workflow.
Credential names may differ, but each node must be mapped to the matching
credential after import.

| Credential | n8n type/use | Used by | Required value |
| --- | --- | --- | --- |
| Tavily Auth | HTTP Header Auth | `Tavily Search` | Tavily API key sent in the header expected by Tavily, normally `X-API-Key` |
| Jina Header | HTTP Header Auth | `Scrape (Jina)` | Jina Reader authentication header, normally `Authorization: Bearer <JINA_API_KEY>` |
| Ollama account | Ollama API | `Ollama Cloud (Analyst)` and `Ollama Cloud (Writer)` | Ollama API endpoint and account/API-key settings required by the n8n credential |

Do not commit API keys or exported secrets. The JSON file contains credential
references, not credentials, but verify the references after importing into a
different n8n instance.

## Import and configure

1. Open n8n and import `Research to Instagram and TikTok Content (Chat).json`
   from the workflow menu.
2. Open the `Tavily Search` node and select your Tavily HTTP Header Auth
   credential.
3. Open `Scrape (Jina)` and select your Jina Header credential. The node already
   sends `X-Return-Format: markdown` and `Accept: text/plain`.
4. Open both Ollama Chat Model nodes and select the same Ollama credential, or
   separate credentials if your deployment requires it.
5. Confirm that the configured Ollama endpoint supports `gemma4:31b`.
6. Open the `Config` node and replace `YOUR COMPANY NAME`.
7. Adjust the brand voice, audience, source thresholds, or source limit if
   needed.
8. Save the workflow and use the Chat Trigger's test URL or activate the
   workflow to obtain its production chat URL.

## Configuration

The values are stored in the `Config` Set node and can be changed without
rewriting the workflow:

| Field | Default | Purpose |
| --- | --- | --- |
| `company_name` | `YOUR COMPANY NAME` | Brand name passed to the writer |
| `brand_voice` | Friendly, practical motorcycle-riding voice | Writing style and safety guidance |
| `audience` | Regular bike enthusiasts | Intended reader/viewer |
| `min_score` | `0.5` | Minimum Tavily result score |
| `max_sources` | `8` | Maximum unique domains sent to scraping |
| `min_sources` | `3` | Minimum independent sources required for a usable result |

The workflow also uses fixed processing limits:

- Tavily requests up to 15 results.
- Jina requests are batched in groups of 3.
- Each usable scraped page contributes at most 10,000 characters.
- The analyst returns at most 12 findings.
- The analyst and writer retry twice after failures.

## Input

Send a plain-language topic through the n8n chat interface. For example:

```text
best chain lubricant for gravel bikes
```

Topics with fewer than five characters receive a prompt to provide a longer
topic.

## Output

For a successful run, the chat response contains:

1. A research briefing grouped into:
   - Universal findings
   - Majority findings
   - Common findings
2. A numbered source list with links; findings include their support counts and
   source IDs.
3. An Instagram caption.
4. An Instagram visual idea.
5. A TikTok spoken script.
6. A TikTok caption.
7. An automated review status.

The writer is instructed to produce:

- Instagram caption: 300–700 characters, with a hook, call to action, and
  5–8 hashtags.
- Instagram visual idea: 1–3 sentences.
- TikTok script: 30–45 seconds and no more than 130 words.
- TikTok caption: no more than 300 characters with 3–5 hashtags.

The final Code node flags over-long captions/scripts, excessive Instagram
hashtags, and numbers that cannot be found in the research briefing. A
successful automated check is not a substitute for human fact review before
publishing.

## Failure and fallback behavior

The workflow deliberately stops instead of generating unsupported content:

| Condition | Chat response |
| --- | --- |
| Topic is shorter than five characters | Requests a usable topic |
| Fewer than three qualifying Tavily candidates | Requests a rephrased or broader topic |
| Fewer than three pages scrape successfully | Says that reliable consensus cannot be established |
| No finding has support from at least three sources | Requests a broader topic |
| Analyst or writer returns invalid/empty JSON | The execution fails visibly so it can be inspected in n8n; it is not converted into an unsupported draft |

Individual Jina scrape errors are allowed to continue so that other sources can
still be evaluated; the minimum successful-source check then determines whether
the run can proceed.

## Design and safety notes

- Source text is treated as data by the prompts; instructions found inside web
  pages are explicitly ignored.
- The analyst uses only claims stated by the retrieved sources and requires
  multi-source agreement.
- The writer is instructed not to invent statistics, quotes, dates, tests, or
  rider stories.
- The brand voice instructs the writer not to glamorize unsafe riding.
- Search results and page contents can change. Review every draft, source, and
  safety-related claim before publishing.
- Respect the terms of service, robots policies, copyright, and privacy
  requirements for every source you retrieve and reuse.

## Troubleshooting

### Credential errors

Open each HTTP Request or Ollama node and confirm that the correct credential is
selected. Imported credential IDs are instance-specific and may not resolve in a
new n8n installation.

### The workflow returns too few sources

Try a more specific or more broadly worded topic, lower `min_score` cautiously,
or increase `max_sources`. A lower score threshold can reduce source quality.

### The workflow returns no consensus

The analyst requires the same finding to be explicitly supported by at least
three sources. Broaden the topic or increase the number of candidate sources;
do not bypass this check merely to force a draft.

### Ollama model errors

Confirm that the Ollama credential can reach the configured endpoint and that
`gemma4:31b` is available. If you change the model, retest both the analyst and
writer nodes because their context sizes and output behavior may differ.

