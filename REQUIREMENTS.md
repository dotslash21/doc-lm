**Requirement Document: LLM-Powered Chat Interface (MVP)**  
**Version**: 1.0  

---

### **1. Objective**  
Enable users to upload **up to 20 plain text documents** (`.txt`), select multiple documents for reference, and interact with an LLM via chat to:  
- Explore, summarize, and explain document content.  
- Receive answers **strictly grounded** in selected documents.  
- Identify conflicting information across documents.  
- Receive explicit warnings when responses are not document-grounded.  

---

### **2. Scope (MVP)**  
#### **In-Scope**  
- **File Support**: `.txt` only.  
- **Limits**:  
  - Max **10MB per file**.  
  - Max **20 documents per session**.  
- **Multi-Document Selection**: Users can select 1+ documents via checkboxes.  
- **Persistence**:  
  - Documents persist indefinitely until manually deleted.  
  - **Shared access**: All users see the same documents/chat history.  
- **Session Management**:  
  - "Session" ends on browser close/refresh.  
  - "Clear history" removes chat messages/selections but retains documents.  

#### **Out-of-Scope**  
- Support for `.docx`, `.pdf`, or images.  
- User authentication or private storage.  
- Automated resolution of conflicting information.  

---

### **3. Functional Requirements**  
#### **3.1 Document Upload & Management**  
- **Upload Interface**:  
  - Drag-and-drop or file picker.  
  - Real-time validation with error messages:  
    - File type: "*Only .txt files are supported*".  
    - File size: "*Maximum file size is 10MB*".  
    - Document count: "*Maximum 20 documents allowed*".  
- **Chunking**:  
  - Split documents into **512-token chunks** (target) with **50-token overlap**.  
  - Prioritize splitting on paragraph boundaries, then sentences.  
- **Storage**:  
  - Store chunks with metadata: `document_title`, `chunk_id`, `embedding`.  
  - Delete documents via UI (removes all associated chunks).  

#### **3.2 Chat Interface**  
- **Document Selection**:  
  - Display document titles with checkboxes in a persistent sidebar.  
  - Selected documents remain active until deselected.  
- **Response Workflow**:  
  1. **Retrieve Chunks**:  
     - Use OpenAI `text-embedding-3-small` for query/chunk embeddings.  
     - Calculate **cosine similarity** between query and chunks.  
     - Prioritize chunks by similarity score.  
     - Include ≥1 chunk per selected document (if relevant).  
  2. **Grounded Response Criteria**:  
     - ✅ Grounded if:  
       - ≥3 chunks with similarity ≥0.75 **OR**  
       - ≥1 chunk with similarity ≥0.85.  
     - ❌ Non-grounded: Show warning banner:  
       "*No relevant content found in the uploaded sources. I’ll attempt an answer, but please verify independently.*"  
  3. **Conflict Handling**:  
     - Present all perspectives with source attribution.  
     - Highlight agreements/disagreements (e.g., "DocumentA states X; DocumentB states Y").  
  4. **LLM Prompt**:  
     ```  
     "Answer using ONLY these excerpts: [chunks].  
     Cite sources like 'DocumentA states: [...]'.  
     Never invent information. Highlight conflicts."  
     ```  
- **Response Formatting**:  
  - **Grounded**: Green message bubble with footer: "*Sources: DocumentA, DocumentB*".  
  - **Non-Grounded**: Yellow message bubble with warning at the top.  

#### **3.3 Error Handling**  
- **Upload Failures**: Block invalid files with error toasts.  
- **Processing Errors**:  
  - Embedding generation: Retry 3 times → "*Document [X] could not be processed*".  
  - LLM timeout: "*Response delayed. Simplify your query*" after 30 seconds.  

---

### **4. Technical Specifications**  
#### **4.1 Backend (FastAPI)**  
- **Embeddings**: OpenAI `text-embedding-3-small` (1536-dim).  
- **Vector Database**: FAISS for chunk storage/retrieval.  
- **Tokenization**: `tiktoken` (cl100k_base) for precise token counting.  
- **Context Management**:  
  - Limit retrieved chunks to 75% of LLM’s context window (e.g., 12k tokens for 16k window).  
- **APIs**:  
  - `POST /upload`: Validate, chunk, and store documents.  
  - `POST /chat`: Process queries, retrieve chunks, generate responses.  
  - `POST /delete`: Remove documents and associated chunks.  

#### **4.2 Frontend (Streamlit)**  
- **UI Components**:  
  - Document uploader with validation.  
  - Sidebar with document titles, checkboxes, and delete buttons.  
  - Chat window with color-coded message bubbles.  
- **Session State**:  
  - Persist selected documents and chat history until page refresh.  

---

### **5. Non-Functional Requirements**  
- **Latency**: Target response time <30 seconds.  
- **Security**: HTTPS for data in transit (no encryption for stored data).  
- **Browser Support**: Latest Chrome, Firefox, Safari.  

---

### **6. Future Iterations**  
- Add direct quotes from documents.  
- Support multi-file formats (`.docx`, `.pdf`).  
- User-specific document isolation.  

---

### **7. Test Scenarios**  
| **Scenario** | **Expected Behavior** |  
|---------------|-----------------------|  
| Upload 21 documents | Block with "*Maximum 20 documents*" |  
| Query with no selected docs | Non-grounded warning + general knowledge answer |  
| Conflicting info in docs | Show all perspectives with source titles |  
| LLM timeout after 30s | Display delay warning |  

---

**Implementation Notes**:  
- Use Streamlit for rapid MVP frontend development.  
- Configure FAISS index with document/chunk metadata.  
- Test tokenization and chunking logic for edge cases.  
