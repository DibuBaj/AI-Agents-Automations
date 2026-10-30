# RAG Support Agent

This project contains two **n8n** workflows for an AI-powered support system:

1. **Support Agent Pre-Processing** - watches a Google Drive folder, reads newly added files, creates embeddings, and stores the document chunks in a Supabase vector table.
2. **Support Agent** - receives a support question through an n8n form, searches the Supabase knowledge base, generates a structured answer with Ollama, and sends the reply through Gmail.

The workflows are designed for the LumiereClip support knowledge base. Update the prompt and form labels if you use the workflows for another product.

## How the system works

```text
Google Drive file
        |
        v
Download file -> Load and split text -> Google Gemini embeddings
                                             |
                                             v
                                      Supabase vector table

Support form -> Ollama AI Agent -> Supabase knowledge search
                         |
                         v
                  PROCEED or IGNORE
                         |
                         v
                    Gmail reply
```

### 1. Knowledge-base preprocessing workflow

The `Support Agent Pre-Processing.json` workflow:

1. Checks a selected Google Drive folder every minute.
2. Starts when a new file is created in that folder.
3. Downloads the file.
4. Loads the binary file as a document.
5. Splits the document into chunks using the Recursive Character Text Splitter.
6. Creates embeddings with Google Gemini.
7. Inserts the chunks and embeddings into the Supabase `documents` table.

Only files added after the trigger is active are processed by this trigger. If you need to index existing files, use an appropriate manual or batch process in n8n.

### 2. Support reply workflow

The `Support Agent.json` workflow:

1. Displays a support form with `Name`, `Email`, and `Question?` fields.
2. Sends the submitted values to the AI Agent.
3. Searches the Supabase vector store before composing an answer.
4. Uses Ollama with the configured `gemma4:31b` model.
5. Keeps a short conversation memory per customer email address.
6. Returns a structured JSON result with:
   - `status`: `PROCEED` or `IGNORE`
   - `reason`
   - `subject`
   - `body`
7. Sends an email only when `status` is `PROCEED`.

The agent instructions tell the model to use only information found in the LumiereClip knowledge base. Spam, gibberish, and unrelated questions are marked `IGNORE` and are not emailed.

## Project files

| File | Description |
|---|---|
| `Support Agent Pre-Processing.json` | n8n workflow for indexing Google Drive files |
| `Support Agent.json` | n8n workflow for answering support form submissions |
| `Support Agent Pre-Processing.png` | Screenshot of the preprocessing workflow |
| `Support Agent.png` | Screenshot of the support workflow |
| `Supabase Vector Database.png` | Screenshot of the populated Supabase vector table |

## Requirements

Before importing the workflows, install or have access to:

- An n8n instance that supports the nodes used in the exports
- A Google Drive account
- A Gmail account
- A Google Cloud project with the Gemini API enabled
- A Supabase project with the `pgvector` extension available
- An Ollama server reachable from n8n
- The Ollama model used by the workflow: `gemma4:31b`

The Ollama server must be reachable from the machine or container running n8n. If n8n runs in Docker and Ollama runs on the host machine, configure the Ollama connection using a hostname reachable from the container rather than assuming `localhost` will work.

## Credentials and configuration

The JSON exports contain placeholder credential IDs. Create the credentials in n8n and select them in the matching nodes after import. Do not put API keys, OAuth client secrets, Supabase service-role keys, or passwords in this repository.

### Credentials required in n8n

| n8n credential type | Used by | What to provide |
|---|---|---|
| Google Drive OAuth2 | `Google Drive Trigger`, `Download file` | OAuth access to the Drive account and the folder containing the support documents |
| Supabase API | Both `Supabase Vector Store` nodes | Supabase project URL and the key required by your n8n/Supabase setup |
| Google Gemini / PaLM API | Both `Embeddings Google Gemini` nodes | A Google AI API key with the required Gemini API access enabled |
| Ollama API | `Ollama Chat Model` | The base URL of the reachable Ollama server |
| Gmail OAuth2 | `Email Reply` | OAuth access to the Gmail account that sends support replies |

### Values to configure in the workflow nodes

After importing the workflows, check these values:

#### Preprocessing workflow

- **Google Drive Trigger → Folder to Watch:** replace `YOUR_GOOGLE_DRIVE_FOLDER_ID` with the folder ID that contains the knowledge-base files.
- **Google Drive nodes:** select the Google Drive OAuth2 credential you created.
- **Supabase Vector Store:** keep or confirm the table name `documents`.
- **Supabase and Gemini nodes:** select the corresponding credentials.
- **Data loader and splitter:** confirm that the selected loader handles the document formats you plan to upload.

#### Support workflow

- **Supabase Vector Store:** replace `YOUR_SUPABASE_TABLE_NAME` with the same table used by preprocessing. The imported preprocessing workflow uses `documents`; the support workflow intentionally contains a placeholder.
- **Ollama Chat Model:** confirm that `gemma4:31b` is installed on the configured Ollama server, or change the model to one you have installed.
- **Gmail node:** select the Gmail OAuth2 credential used to send replies.
- **Google Gemini embeddings:** use the same embedding configuration as the preprocessing workflow. The query embedding configuration must be compatible with the embeddings stored in Supabase.
- **Form fields:** keep the field names unchanged unless you also update the AI Agent expressions:
  - `Name`
  - `Email`
  - `Question?`

## Supabase vector table

The two workflows must use the same Supabase vector table and a compatible embedding configuration.

The preprocessing workflow is configured to insert into a table named `documents`. The support workflow reads from that table as a retrieval tool after you replace its `YOUR_SUPABASE_TABLE_NAME` placeholder.

Before running the workflows:

1. Create or select the Supabase project.
2. Enable the `vector`/`pgvector` extension if it is not already enabled.
3. Create the vector-store table using the schema expected by the n8n Supabase Vector Store node.
4. Make sure the embedding column dimension matches the Google Gemini embedding model configured in n8n.
5. Confirm that the credentials used by n8n can read and insert rows in the table.

Do not guess the vector dimension: confirm it from the embedding model selected in your Google Gemini credential/node and the current n8n/Supabase vector-store setup.

After indexing a file, verify that rows contain document content, metadata, and vector embeddings. The included screenshot shows the expected populated-table view:

![Supabase vector database](./Supabase%20Vector%20Database.png)

## Import and setup

1. Open your n8n instance.
2. Import `Support Agent Pre-Processing.json` as a workflow.
3. Import `Support Agent.json` as a second workflow.
4. Create the required credentials listed above.
5. Assign each credential to the matching imported node.
6. Set the Google Drive folder ID in the preprocessing workflow.
7. Confirm that both vector-store nodes use the same Supabase table.
8. Confirm the embedding configuration is compatible in both workflows.
9. Confirm that the Ollama server has the configured model:

   ```text
   gemma4:31b
   ```

10. Save the workflows.
11. Run the preprocessing workflow with a test knowledge-base file and check Supabase.
12. Activate the preprocessing workflow after the test succeeds.
13. Open the support workflow's form URL and submit a test question.
14. Check the n8n execution log and the Gmail mailbox for the result.
15. Activate the support workflow after the test succeeds.

Both exported workflows are currently inactive. Importing them does not automatically make them live.

## Screenshots

### Preprocessing workflow

![Support Agent Pre-Processing workflow](./Support%20Agent%20Pre-Processing.png)

### Support workflow

![Support Agent workflow](./Support%20Agent.png)

## Testing checklist

- [ ] A new file in the configured Drive folder starts the preprocessing workflow.
- [ ] The file is downloaded successfully.
- [ ] New rows appear in the Supabase vector table.
- [ ] The rows contain content, metadata, and embeddings.
- [ ] The support form accepts a name, valid email address, and question.
- [ ] A knowledge-base question produces `PROCEED` and sends one email.
- [ ] An unrelated or invalid message produces `IGNORE` and does not send an email.
- [ ] The email is sent from the intended Gmail account.
- [ ] n8n execution logs show no credential, embedding, or vector-store errors.

## Important notes

- Keep all API keys and OAuth secrets in n8n credentials or a secure secret manager.
- Do not commit exported workflows containing real credential values.
- The support agent is instructed to answer only from the indexed knowledge base. Keep that knowledge base current.
- Changing the embedding model after documents are indexed may make existing vectors incompatible. Re-index the knowledge base when changing embedding dimensions or configuration.
- The support workflow uses the customer's email as its memory key. Treat submitted email addresses as personal data and apply your organization's retention and privacy policies.
- The Gmail node sends plain-text email and has attribution disabled in the exported workflow.
- Review the AI-generated email behavior before enabling the workflow for real customers.

## Known values that must be supplied by the installer

These values are intentionally not included in the repository:

- Google Drive folder ID
- n8n credential IDs and credential names
- Supabase project URL and key
- Supabase vector table name if it is not `documents`
- Google Gemini API key
- Ollama server URL
- Gmail OAuth configuration
- The public n8n form URL after the support workflow is imported

