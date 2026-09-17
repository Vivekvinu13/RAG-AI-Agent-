# Telegram RAG AI Agent with Google Drive, Pinecone, and External Tools

## Overview

This n8n workflow implements a Retrieval-Augmented Generation (RAG)
assistant accessible through Telegram.

The workflow has two primary sections:

1.  **Knowledge ingestion pipeline**: Monitors Google Drive, downloads
    newly created files, loads the document content, generates OpenAI
    embeddings, and stores the vectors in Pinecone.
2.  **Telegram question-answering pipeline**: Receives Telegram
    messages, processes them with an AI Agent, retrieves relevant
    information from the Pinecone vector store, and responds through
    Telegram.

The AI Agent is also connected to additional tools, including:

-   Airtable record retrieval
-   Gmail messaging
-   Amazon search through SerpApi
-   A vector-store question-answering tool
-   OpenAI Chat Models
-   Simple Memory

------------------------------------------------------------------------

## Architecture

``` text
                    KNOWLEDGE INGESTION PIPELINE

Google Drive Trigger
        |
        v
   Download File
        |
        v
Pinecone Vector Store
   |             |
   |             +--> OpenAI Embeddings
   |
   +-----------------> Default Data Loader


                    TELEGRAM AI ASSISTANT

Telegram Trigger
        |
        v
     AI Agent
        |
        v
Send a Text Message

AI Agent connections:
- OpenAI Chat Model
- Simple Memory
- Answer Questions with a Vector Store
- Send a Message in Gmail
- Get a Record in Airtable
- Amazon Search in SerpApi

Vector-store QA tool:
- Pinecone Vector Store
- OpenAI Embeddings
- OpenAI Chat Model
```

------------------------------------------------------------------------

## Workflow Objectives

The workflow is designed to:

-   Ingest documents from Google Drive.
-   Convert document content into vector embeddings.
-   Store embeddings in Pinecone for semantic retrieval.
-   Allow users to ask questions through Telegram.
-   Retrieve relevant information from indexed documents.
-   Generate grounded answers using an AI Agent.
-   Maintain conversational context through memory.
-   Execute additional actions using connected tools.

------------------------------------------------------------------------

# 1. Knowledge Ingestion Pipeline

## 1.1 Google Drive Trigger

**Node:** `Google Drive Trigger`

The Google Drive Trigger starts the ingestion pipeline when a file is
created in the configured Google Drive location.

### Responsibilities

-   Monitor Google Drive for newly created files.
-   Start the ingestion process automatically.
-   Pass file metadata to the download step.

### Configuration considerations

-   Configure Google Drive credentials.
-   Select the appropriate event, such as `fileCreated`.
-   Restrict monitoring to the required folder or location where
    supported.
-   Confirm that the workflow can access the relevant file types.

------------------------------------------------------------------------

## 1.2 Download File

**Node:** `Download file`

Downloads the file identified by the Google Drive Trigger.

The downloaded file is passed to the document-loading stage so that its
content can be processed.

### Configuration considerations

-   Map the file ID from the trigger.
-   Confirm that the file is downloaded in a format supported by the
    data loader.
-   Handle missing files, inaccessible files, and permission errors.
-   Validate the downloaded file before ingestion.

------------------------------------------------------------------------

## 1.3 Default Data Loader

**Node:** `Default Data Loader`

Loads the downloaded file and converts it into document objects that can
be embedded.

Depending on the file type and configuration, the loader may extract:

-   Text
-   Document metadata
-   File name
-   Source information
-   Page or section details, where supported

The resulting documents are passed to the embedding and vector-store
pipeline.

### Recommended practices

-   Preserve source metadata for traceability.
-   Split large documents into appropriate chunks.
-   Avoid excessively large chunks that reduce retrieval precision.
-   Use consistent metadata fields such as source file ID, file name,
    folder, and ingestion timestamp.

------------------------------------------------------------------------

## 1.4 OpenAI Embeddings

**Node:** `Embeddings OpenAI`

Generates numerical vector representations of document chunks.

Embeddings allow semantically similar content to be identified even when
the user's wording differs from the original document.

### Responsibilities

-   Convert document chunks into embedding vectors.
-   Use the same embedding model during ingestion and retrieval.
-   Support semantic similarity search in Pinecone.

### Important requirement

The embedding model used during document ingestion should be compatible
with the embedding model used by the question-answering retrieval
workflow.

------------------------------------------------------------------------

## 1.5 Pinecone Vector Store

**Node:** `Pinecone Vector Store`

Stores the document embeddings and associated content or metadata in
Pinecone.

The vector store supports semantic retrieval when the AI Agent needs
information from the indexed documents.

### Typical stored information

-   Embedding vector
-   Document text or reference
-   Source metadata
-   File name
-   Document ID
-   Chunk identifier
-   Ingestion timestamp

### Configuration considerations

-   Configure the Pinecone credentials.
-   Select the correct index and namespace.
-   Ensure the vector dimensions match the selected embedding model.
-   Define appropriate metadata fields.
-   Prevent duplicate ingestion where possible.
-   Plan for document updates and deletions.

------------------------------------------------------------------------

# 2. Telegram AI Assistant Pipeline

## 2.1 Telegram Trigger

**Node:** `Telegram Trigger`

Receives incoming Telegram messages and starts the question-answering
workflow.

Users can ask questions about the documents indexed in Pinecone or
request actions using the connected tools.

### Possible requests

-   Ask questions about uploaded documents.
-   Request a summary of a document.
-   Search for information in the knowledge base.
-   Retrieve an Airtable record.
-   Send an email.
-   Search Amazon products.
-   Ask follow-up questions using conversation memory.

------------------------------------------------------------------------

## 2.2 AI Agent

**Node:** `AI Agent`

The AI Agent is the central orchestration component of the Telegram
assistant.

It is responsible for:

1.  Understanding the incoming user message.
2.  Determining whether the request is knowledge-related or
    action-oriented.
3.  Using the OpenAI Chat Model to reason and generate responses.
4.  Accessing conversation memory when appropriate.
5.  Calling the vector-store question-answering tool for document-based
    questions.
6.  Calling Gmail, Airtable, or SerpApi tools when required.
7.  Returning a final response to the Telegram node.

The AI Agent should be configured with clear instructions about when to
use each tool and how to handle uncertain results.

------------------------------------------------------------------------

## 2.3 OpenAI Chat Model

**Node:** `OpenAI Chat Model`

Provides the language model used by the AI Agent.

### Responsibilities

-   Interpret natural-language requests.
-   Identify user intent.
-   Decide when a tool is needed.
-   Generate responses based on retrieved information.
-   Summarize search or tool results.
-   Ask clarification questions when required.

The selected model, system prompt, temperature, token limits, and
credentials should be maintained in the n8n node configuration.

------------------------------------------------------------------------

## 2.4 Simple Memory

**Node:** `Simple Memory`

Maintains conversation context for the Telegram assistant.

Memory can help the AI Agent:

-   Understand follow-up questions.
-   Refer to earlier messages.
-   Maintain context during a conversation.
-   Reduce the need for users to repeat information.

### Multi-user considerations

For deployments serving multiple Telegram users, memory sessions should
be separated using a stable identifier such as the Telegram chat ID or
user ID. This helps prevent context from one user being exposed to
another user.

------------------------------------------------------------------------

## 2.5 Send a Text Message

**Node:** `Send a text message`

Sends the AI Agent's final response to the Telegram user.

The response may contain:

-   A direct answer.
-   A document-grounded answer.
-   A summary of retrieved content.
-   Search results.
-   Confirmation of a completed action.
-   A clarification request.
-   A user-friendly error message.

The Telegram chat ID and response content should be mapped correctly
from the workflow execution data.

------------------------------------------------------------------------

# 3. Retrieval-Augmented Generation (RAG) Tool

## 3.1 Answer Questions with a Vector Store

**Node:** `Answer questions with a vector store`

This tool provides the AI Agent with a retrieval-based
question-answering capability.

When the user asks a question about indexed documents, the tool can:

1.  Convert the user's question into an embedding.
2.  Search the Pinecone vector store for relevant document chunks.
3.  Retrieve matching content.
4.  Pass the retrieved context to an OpenAI Chat Model.
5.  Generate an answer based on the retrieved information.
6.  Return the result to the AI Agent.

This architecture is commonly known as **Retrieval-Augmented Generation
(RAG)**.

------------------------------------------------------------------------

## 3.2 Retrieval Tool Dependencies

The vector-store question-answering tool is connected to:

  Component               Purpose
  ----------------------- ---------------------------------------------
  Pinecone Vector Store   Retrieves relevant document chunks
  OpenAI Embeddings       Converts the user query into a vector
  OpenAI Chat Model       Generates an answer using retrieved context

### Retrieval quality considerations

-   Use the same embedding model for ingestion and retrieval.
-   Select an appropriate chunk size and overlap.
-   Preserve useful document metadata.
-   Tune the number of retrieved chunks.
-   Evaluate retrieval relevance using representative questions.
-   Add source references where possible.
-   Instruct the model not to invent information when the retrieved
    context is insufficient.

------------------------------------------------------------------------

# 4. Connected AI Agent Tools

## 4.1 Send a Message in Gmail

**Node:** `Send a message in Gmail`

Allows the AI Agent to send messages through Gmail.

### Potential use cases

-   Sending a document summary.
-   Communicating retrieved information.
-   Sending a notification.
-   Sending an email based on a Telegram request.

### Recommended safeguards

-   Confirm the recipient when the request is ambiguous.
-   Confirm the subject and message body for sensitive messages.
-   Use least-privilege Gmail permissions.
-   Add explicit confirmation before sending high-impact or external
    communications.

------------------------------------------------------------------------

## 4.2 Get a Record in Airtable

**Node:** `Get a record in Airtable`

Retrieves a record from a configured Airtable base or table.

### Potential use cases

-   Looking up customer information.
-   Retrieving project records.
-   Checking task or workflow status.
-   Fetching structured business data.

### Configuration considerations

-   Configure Airtable credentials.
-   Specify the correct base and table.
-   Map the record identifier or search criteria.
-   Restrict access to authorized data.
-   Handle missing records and permission failures.

------------------------------------------------------------------------

## 4.3 Amazon Search in SerpApi

**Node:** `Amazon search in SerpApi`

Allows the AI Agent to search Amazon-related product information through
SerpApi.

### Potential use cases

-   Searching for products.
-   Finding product listings.
-   Comparing product information.
-   Supporting shopping-related questions.

### Considerations

-   Configure SerpApi credentials.
-   Use suitable search parameters and region settings.
-   Treat search results as retrieved information rather than guaranteed
    current availability.
-   Clearly distinguish search results from verified price, stock, or
    delivery information.

------------------------------------------------------------------------

# 5. End-to-End Execution

## 5.1 Document Ingestion Flow

1.  A file is created in Google Drive.
2.  The Google Drive Trigger detects the new file.
3.  The Download File node retrieves the file.
4.  The Default Data Loader extracts the document content.
5.  OpenAI Embeddings converts document chunks into vectors.
6.  Pinecone Vector Store stores the vectors and metadata.
7.  The document becomes available for semantic retrieval.

## 5.2 Telegram Question-Answering Flow

1.  A user sends a message through Telegram.
2.  Telegram Trigger receives the message.
3.  The AI Agent interprets the request.
4.  The OpenAI Chat Model supports reasoning and response generation.
5.  Simple Memory provides conversation context when needed.
6.  For knowledge-base questions, the AI Agent invokes the vector-store
    QA tool.
7.  Pinecone retrieves relevant document chunks.
8.  OpenAI Embeddings and the retrieval model support semantic search
    and answer generation.
9.  For action requests, the AI Agent may call Gmail, Airtable, or
    SerpApi.
10. The AI Agent prepares the final response.
11. The Send a Text Message node replies to the Telegram user.

------------------------------------------------------------------------

# 6. Example User Requests

  -----------------------------------------------------------------------
  User request                        Expected component
  ----------------------------------- -----------------------------------
  "Summarize the document I           Vector-store QA tool
  uploaded."                          

  "What does the policy say about     Pinecone retrieval and OpenAI Chat
  approvals?"                         Model

  "Find the record for customer ABC." Airtable

  "Send the summary to my manager."   Gmail

  "Search Amazon for a laptop under   SerpApi
  my budget."                         

  "What did we discuss earlier?"      Simple Memory

  "Which document contains this       Vector retrieval and document
  information?"                       metadata
  -----------------------------------------------------------------------

Actual behavior depends on the AI Agent prompt, tool configuration,
permissions, and indexed content.

------------------------------------------------------------------------

# 7. Required Integrations

  Integration        Purpose
  ------------------ ------------------------------------------
  Google Drive       Detect and download new documents
  OpenAI             Chat model and embeddings
  Pinecone           Store and retrieve document vectors
  Telegram Bot API   Receive user messages and send responses
  Gmail              Send email messages
  Airtable           Retrieve structured records
  SerpApi            Search Amazon product information
  n8n                Workflow orchestration

------------------------------------------------------------------------

# 8. Configuration Checklist

## Google Drive and Ingestion

-   [ ] Configure Google Drive credentials.
-   [ ] Configure the file-created trigger.
-   [ ] Map the file ID to the download node.
-   [ ] Configure the Default Data Loader.
-   [ ] Confirm supported file types.
-   [ ] Configure document chunking and metadata.
-   [ ] Connect OpenAI Embeddings.
-   [ ] Configure Pinecone index and namespace.
-   [ ] Validate vector dimensions.
-   [ ] Test a complete document ingestion.

## Telegram and AI Agent

-   [ ] Configure Telegram bot credentials.
-   [ ] Configure the Telegram Trigger.
-   [ ] Connect the OpenAI Chat Model.
-   [ ] Configure the AI Agent system prompt.
-   [ ] Configure Simple Memory and session IDs.
-   [ ] Connect the vector-store QA tool.
-   [ ] Connect Gmail, Airtable, and SerpApi tools as required.
-   [ ] Map the AI Agent output to Telegram.
-   [ ] Test normal questions and tool-based requests.

------------------------------------------------------------------------

# 9. Security and Governance

Because the AI Agent can access documents and perform external actions,
appropriate controls should be implemented.

## Credentials

-   Store credentials in n8n's credential manager.
-   Do not hardcode API keys or access tokens.
-   Use least-privilege permissions.
-   Separate development and production credentials where possible.

## Data Access

-   Restrict Pinecone namespaces and document access appropriately.
-   Ensure users only retrieve information they are authorized to view.
-   Separate memory sessions by Telegram user or chat.
-   Avoid exposing confidential document content in Telegram responses.

## External Actions

Use confirmation steps before:

-   Sending emails.
-   Updating or exposing business information.
-   Performing actions involving external recipients.
-   Accessing sensitive Airtable records.

## RAG Governance

-   Preserve document source metadata.
-   Provide citations or source identifiers where possible.
-   Instruct the AI Agent to acknowledge when information is not found.
-   Avoid presenting unsupported assumptions as facts.
-   Define document update and deletion procedures.

------------------------------------------------------------------------

# 10. Error Handling

The workflow should handle the following scenarios:

-   Google Drive trigger failures.
-   File download failures.
-   Unsupported document formats.
-   Data-loader extraction errors.
-   OpenAI API failures.
-   Embedding generation failures.
-   Pinecone connection or indexing errors.
-   Empty or irrelevant retrieval results.
-   Gmail authentication failures.
-   Airtable record not found.
-   SerpApi failures or no search results.
-   Telegram delivery failures.

Recommended error-handling steps:

1.  Capture the technical error securely.
2.  Log relevant execution information.
3.  Return a clear user-facing response.
4.  Avoid exposing credentials or internal stack traces.
5.  Retry only transient failures when safe.
6.  Alert an administrator for repeated or critical failures.

------------------------------------------------------------------------

# 11. Testing Strategy

  -----------------------------------------------------------------------
  Test scenario                       Expected result
  ----------------------------------- -----------------------------------
  New Google Drive file               File is downloaded and indexed

  Unsupported file type               Workflow reports a controlled error

  Duplicate file                      Duplicate ingestion is prevented or
                                      detected

  Basic document question             Relevant context is retrieved

  Question with no matching content   Agent acknowledges insufficient
                                      information

  Follow-up question                  Memory provides relevant context

  Gmail request                       Email is sent only with valid
                                      authorization

  Airtable lookup                     Correct record is retrieved

  Amazon search                       SerpApi returns and summarizes
                                      results

  Multiple Telegram users             Memory and access remain isolated

  Pinecone outage                     User receives a controlled fallback
                                      response

  Telegram failure                    Delivery issue is logged
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 12. Monitoring and Metrics

Recommended metrics include:

## Ingestion Metrics

-   Number of files detected.
-   Number of files successfully downloaded.
-   Number of documents indexed.
-   Number of ingestion failures.
-   Average ingestion processing time.
-   Duplicate ingestion rate.
-   Number of chunks generated per document.

## Retrieval Metrics

-   Retrieval latency.
-   Number of retrieved chunks.
-   Retrieval relevance.
-   No-result rate.
-   Answer groundedness.
-   User feedback on answer quality.
-   Citation or source-reference coverage.

## Agent Metrics

-   AI Agent response latency.
-   Tool invocation count.
-   Tool success and failure rate.
-   Clarification request rate.
-   OpenAI token usage and cost.
-   Telegram response delivery success rate.

------------------------------------------------------------------------

# 13. Suggested Enhancements

-   Add document update and deletion synchronization.
-   Add duplicate detection using file IDs or content hashes.
-   Store document version information.
-   Add source citations to RAG responses.
-   Add access control based on Telegram user identity.
-   Add hybrid search using keyword and vector retrieval.
-   Add reranking for improved retrieval quality.
-   Add metadata filtering by document type, department, or access
    level.
-   Add persistent conversation storage.
-   Add human approval for sensitive external actions.
-   Add automated retrieval evaluation datasets.
-   Add retry logic with exponential backoff.
-   Add centralized logging and alerting.
-   Add monitoring for latency, cost, and retrieval quality.

------------------------------------------------------------------------

# 14. Summary

This n8n workflow implements a Telegram-based RAG AI assistant with
automated Google Drive ingestion.

The ingestion pipeline:

-   Detects new Google Drive files.
-   Downloads and loads documents.
-   Generates OpenAI embeddings.
-   Stores vectors in Pinecone.

The Telegram assistant:

-   Receives user questions.
-   Uses an OpenAI Chat Model and Simple Memory.
-   Retrieves document context through a Pinecone vector store.
-   Generates answers using a vector-store question-answering tool.
-   Supports additional tools such as Gmail, Airtable, and Amazon
    search.
-   Sends the final response back through Telegram.

The solution combines **document ingestion, vector databases, semantic
search, Retrieval-Augmented Generation, AI Agent orchestration, and
external business tools** in an extensible n8n workflow.
