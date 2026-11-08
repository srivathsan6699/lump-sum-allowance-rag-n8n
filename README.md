# Lump Sum Allowance Q&A (n8n RAG)

An n8n workflow that answers questions about the UK lump sum allowance using retrieval-augmented generation (RAG). A document is chunked, embedded and stored in Pinecone. A chat agent then retrieves the relevant passages and answers only from them, with a source line taken from the retrieved passage.

Built by Srivathsan Suresh.

> This is a learning and portfolio project. It is not financial or pension advice.

Source document: HMRC Pensions Tax Manual, PTM171000, "Lump sum allowance and lump sum and death benefit allowance: lump sum allowance" (GOV.UK): https://www.gov.uk/hmrc-internal-manuals/pensions-tax-manual/ptm171000

The PDF itself is not included in this repo. Download the page from GOV.UK and upload it through the ingestion form.

## Architecture

Two workflows, in `workflows/`. Add a screenshot of your n8n canvas as `docs/workflow-screenshot.png`.

```
Ingestion (ingest.json)
  Form submission (PDF upload) -> Pinecone Vector Store (insert)
                                    |-- Embeddings (OpenAI)
                                    |-- Document loader (Default Data Loader, PDF, adds a "source" field)

Chat (chat-agent.json)
  Chat message received -> AI Agent
                             |-- Chat model (OpenAI gpt-4o-mini, temperature 0)
                             |-- Memory (Simple Memory)
                             |-- Tool: "Search lump sum allowance document"
                                   Pinecone Vector Store, retrieve as tool, top-K 4
                                   |-- Embeddings (OpenAI)
```

The search tool hands the retrieved passages and their metadata straight to the agent. An earlier version used a "vector store question answer" tool with its own second chat model, which dropped the metadata, so the agent could not cite anything.

## Settings used

| Setting | Value |
| --- | --- |
| Embedding model | OpenAI Embeddings node, model option left at the node default on both ingestion and chat (confirm the name in n8n and write it here) |
| Chat model | gpt-4o-mini, temperature 0 |
| Chunking | Default Data Loader with PDF loader, splitting options left at defaults (confirm the values in n8n and write them here) |
| Pinecone index / namespace | `n8nrag-vectors` / no namespace set |
| Top-K retrieved chunks | 4 |

## Setup

1. Install and open n8n (cloud or self-hosted).
2. Import the files in `workflows/` (Workflows, then Import from file).
3. Add your own credentials in n8n under Credentials: OpenAI and Pinecone. Never paste keys into node fields.
4. Create a Pinecone index whose dimension matches your embedding model, and select it in both Pinecone nodes.
5. Run the ingestion workflow and upload the PDF once. If you upload again, empty the index first, otherwise you get duplicate chunks that waste retrieval slots.
6. Open the chat workflow and ask a question.

## Keeping answers on topic

- The agent has a system message (see `prompts/system-message.md`) telling it to answer only from retrieved text, refuse anything unrelated, and end with a source line.
- Source lines use only what the retrieved passage contains: the document title, the line range, and any legislation reference written in the passage. Chunks do not carry page numbers, so the prompt forbids stating one.
- Behaviour is checked against a fixed question set in `tests/test-questions.md`.

## Test results

Smoke test on 8 November 2026 (3 runs, not the full set):

- "What is the lump sum allowance?" returned an answer matching the retrieved passage, with a source line giving the document title, lines 34 to 56, and the Section 637P ITEPA 2003 reference from the passage.
- "Give me Python code for a palindrome checker" returned the fixed refusal message and did not call the search tool.
- An earlier version of the prompt asked for page numbers and produced a page citation that was not backed by the passage. That is why the citation rule changed.

The full question set in `tests/test-questions.md` has not been run yet. Add the totals here when it has.

## Known limitations

- Language models can be unreliable at arithmetic. Calculations should go through a code tool rather than the model.
- Answers are only as good as the ingested document. If the rules change, re-ingest the new version.
- Chunks do not store page numbers, so citations point to the document title and line range.
- Uploading the same document twice creates duplicate chunks. Empty the index before re-uploading.
- Simple Memory is held in n8n's memory and resets on restart.

## Security

- No API keys, tokens or personal data are stored in this repo.
- Credentials live in n8n's credential store. Use `.env.example` as a template for any environment variables.
- Webhook IDs and credential IDs were removed from the exported workflows. n8n creates new ones on import, and you reconnect your own OpenAI and Pinecone credentials.
- The chat trigger is set to non-public. If you make it public and activate the workflow, anyone with the chat URL can use your OpenAI and Pinecone credits. The upload form, if activated, can write to your index.

## Licence

MIT. See `LICENSE`.
