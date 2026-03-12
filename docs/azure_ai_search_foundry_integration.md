
# Azure AI Search and Foundry Integration – Configuration Guide (captured for workshop transcript)

## Step-by-Step Configuration Instructions

1. **Provision Azure AI Search Service**
   - Create Azure Cognitive Search (e.g., "AI Search 0310")
   - Enable Azure RBAC (disable API key-only access)

2. **Create and Configure Search Index**
   - Define index schema
   - Configure cognitive skills or vector search if needed
   - Ensure authentication uses managed identity

3. **Assign RBAC Roles to Foundry Managed Identity**
   - Assign `Search Index Data Contributor` and `Search Index Data Reader`
   - Scope: Azure Search Resource

4. **Assign RBAC Roles to User (e.g., AI Developer)**
   - Assign `Search Service Contributor` to allow UI integration
   - Optionally assign `Search Index Data Contributor` or `Reader`

5. **Connect Azure Search to Foundry Agent**
   - Add Azure Search as a knowledge source in Foundry
   - Select index and embedding model (if applicable)
   - Troubleshoot visibility and permission issues

6. **Test and Troubleshoot Integration**
   - Query agent and validate results
   - Check indexer logs and vectorization errors
   - Ensure correct embedding model and retrievable fields

## Best Practices and Lessons Learned

- Always use **managed identities** (keyless auth)
- Verify authentication settings in Azure portal
- Assign roles to the correct identity (user vs. managed identity)
- Use least privilege: assign roles at the resource level
- Ensure Azure Search is configured for RBAC access

## Azure RBAC Role Assignment Table

| Persona / Resource Identity         | Required Role(s)                          | Scope Level         | Purpose / Notes                                                                 |
|------------------------------------|-------------------------------------------|---------------------|---------------------------------------------------------------------------------|
| Foundry Project Managed Identity   | Search Index Data Contributor             | Azure Search Resource | To allow indexing and document ingestion into the search index                 |
| Foundry Project Managed Identity   | Search Index Data Reader                  | Azure Search Resource | To allow querying the index from Foundry agent                                 |
| Foundry Project Managed Identity   | Search Service Contributor (optional)     | Azure Search Resource | Needed if Foundry needs to manage the search service itself                    |
| User (e.g. Babar Batla)            | Search Service Contributor                | Azure Search Resource | To allow user to connect and manage search service from Foundry UI             |
| User (e.g. Babar Batla)            | Search Index Data Contributor (optional)  | Azure Search Resource | To allow user to create indexes or upload documents directly                   |
| User (e.g. Babar Batla)            | Search Index Data Reader (optional)       | Azure Search Resource | To allow user to browse/query indexes directly                                 |
| Azure Search Service Identity      | AI User / Cognitive Services User         | Azure OpenAI Resource | If using vector embeddings from Azure OpenAI, search service needs access      |
| Foundry Project Managed Identity   | Storage Blob Data Contributor             | Azure Storage Resource | To allow reading/writing documents to blob storage for indexing                |

