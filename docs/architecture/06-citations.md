# Citations

## Overview

Citations link AI answers back to source documents. Critical for trust and verification in legal contexts. Users must be able to click a citation and see the exact source.

## Citation Model

```typescript
interface Citation {
  index: number;                // [1], [2], etc. as shown in answer
  
  // Document reference
  documentId: string;
  documentTitle: string;
  documentType: 'document' | 'email';
  
  // Location within document
  chunkId: string;
  location: CitationLocation;
  
  // Quoted content
  snippet: string;              // The relevant passage
  
  // Deep link
  webUrl: string;               // Link to open in source system
  deepLink?: string;            // Link with location hint if supported
}

interface CitationLocation {
  // For documents
  page?: number;
  section?: string;
  heading?: string;
  paragraph?: number;
  
  // Character offsets (for highlighting)
  charStart?: number;
  charEnd?: number;
  
  // For emails
  emailPosition?: 'subject' | 'body' | 'attachment';
  attachmentName?: string;
}
```

## Citation Generation

### During RAG

Citations are tracked through the RAG pipeline:

```typescript
// 1. Each chunk gets a citation index during context building
const context = buildContext(chunks); // Assigns citationIndex 1, 2, 3...

// 2. Prompt instructs LLM to use citation format
const prompt = `...Cite sources using [1], [2], etc...`;

// 3. Parse citations from LLM response
function extractCitations(
  answer: string,
  contextChunks: ContextChunk[]
): Citation[] {
  // Find all [N] references in the answer
  const citationPattern = /\[(\d+)\]/g;
  const usedIndices = new Set<number>();
  
  let match;
  while ((match = citationPattern.exec(answer)) !== null) {
    usedIndices.add(parseInt(match[1]));
  }
  
  // Build citations only for referenced chunks
  return contextChunks
    .filter(chunk => usedIndices.has(chunk.citationIndex))
    .map(chunk => ({
      index: chunk.citationIndex,
      documentId: chunk.documentId,
      documentTitle: chunk.documentTitle,
      documentType: chunk.documentType,
      chunkId: chunk.id,
      location: chunk.metadata,
      snippet: truncate(chunk.content, 200),
      webUrl: chunk.webUrl,
      deepLink: buildDeepLink(chunk),
    }));
}
```

### Storing Location Metadata

Location data is captured during ingestion:

```typescript
// During chunking
interface ChunkMetadata {
  // Source document
  documentId: string;
  documentTitle: string;
  webUrl: string;
  
  // Location
  page?: number;              // For PDFs: page number
  heading?: string;           // Nearest heading above chunk
  section?: string;           // Section title if available
  charStart: number;          // Character offset in original
  charEnd: number;
  
  // For emails
  emailId?: string;
  isAttachment?: boolean;
  attachmentIndex?: number;
}

// Example: PDF chunking with page tracking
function chunkPdf(content: PdfContent): Chunk[] {
  const chunks: Chunk[] = [];
  
  for (const page of content.pages) {
    const pageChunks = splitIntoChunks(page.text, {
      targetSize: 500,
      overlap: 50,
    });
    
    for (const chunk of pageChunks) {
      chunks.push({
        content: chunk.text,
        metadata: {
          page: page.number,
          charStart: chunk.start,
          charEnd: chunk.end,
        },
      });
    }
  }
  
  return chunks;
}
```

## Deep Linking

### M365 Deep Links

Different formats for different content types:

```typescript
function buildDeepLink(chunk: ContextChunk): string {
  const { documentType, webUrl, metadata } = chunk;
  
  switch (documentType) {
    case 'document':
      // SharePoint/OneDrive - can hint at page for PDFs
      if (metadata.page && webUrl.includes('.pdf')) {
        return `${webUrl}#page=${metadata.page}`;
      }
      // Word docs - no deep link to paragraph, just open doc
      return webUrl;
      
    case 'email':
      // Outlook web - deep link to specific message
      return `https://outlook.office.com/mail/deeplink/read/${chunk.emailId}`;
      
    default:
      return webUrl;
  }
}
```

### Deep Link Limitations

| Content Type | Deep Link Support | Workaround |
|--------------|-------------------|------------|
| PDF | Page number (#page=N) | Good support |
| Word/DOCX | No paragraph linking | Show snippet for user to Ctrl+F |
| Excel | Sheet name possible | Limited |
| PowerPoint | Slide number possible | Limited |
| Email | Message ID works | Good support |
| Email attachment | No direct link | Link to email + attachment name |

### Citation UI Hint

Since deep linking to specific paragraphs isn't always possible, include the snippet:

```typescript
interface CitationDisplay {
  index: number;
  title: string;
  snippet: string;            // "Payment shall be due within 30 days..."
  snippetHighlight: string;   // Keywords to highlight
  location: string;           // "Page 5" or "Section 3.2"
  action: {
    label: string;            // "Open in SharePoint"
    url: string;
    hint?: string;            // "Search for: payment terms"
  };
}
```

## Citation in Chat UI

### Display Format

```
┌─────────────────────────────────────────────────────────────┐
│  AI Response:                                                │
│                                                             │
│  According to the Services Agreement [1], payment is due    │
│  within 30 days of invoice date. The agreement also         │
│  specifies a 1.5% monthly late fee [1]. For the specific    │
│  project milestones, see the Statement of Work [2].         │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  Sources:                                                    │
│                                                             │
│  [1] Acme Corp - Services Agreement.pdf                     │
│      📄 Page 5 · "Payment shall be due within 30 days..."   │
│      [Open in SharePoint →]                                 │
│                                                             │
│  [2] Acme Corp - Statement of Work.docx                     │
│      📄 Section 4 · "Project milestones are defined as..."  │
│      [Open in SharePoint →]                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Inline Citation Hover

When user hovers over [1], show preview:

```typescript
interface CitationPreview {
  title: string;
  snippet: string;
  location: string;
  thumbnail?: string;         // Document preview image (future)
}
```

## Citation Verification

### Hallucination Detection

LLMs may cite sources incorrectly. Mitigation:

```typescript
async function verifyCitations(
  answer: string,
  citations: Citation[],
  contextChunks: ContextChunk[]
): Promise<VerificationResult> {
  const issues: CitationIssue[] = [];
  
  for (const citation of citations) {
    const chunk = contextChunks.find(c => c.citationIndex === citation.index);
    
    if (!chunk) {
      issues.push({
        type: 'missing_source',
        citationIndex: citation.index,
        message: 'Citation references non-existent source',
      });
      continue;
    }
    
    // Check if the claim actually appears in the source
    // (simplified - could use embeddings for semantic match)
    const claimNearCitation = extractClaimNearCitation(answer, citation.index);
    const similarity = computeSimilarity(claimNearCitation, chunk.content);
    
    if (similarity < 0.5) {
      issues.push({
        type: 'weak_support',
        citationIndex: citation.index,
        message: 'Claim may not be fully supported by cited source',
        similarity,
      });
    }
  }
  
  return {
    valid: issues.length === 0,
    issues,
  };
}
```

### User Feedback

Allow users to report bad citations:

```typescript
interface CitationFeedback {
  citationId: string;
  userId: string;
  feedbackType: 'incorrect' | 'misleading' | 'helpful';
  comment?: string;
  timestamp: Date;
}
```

## Citation Analytics

Track citation usage for quality improvement:

```typescript
interface CitationMetrics {
  // Per document
  documentId: string;
  citedCount: number;          // Times cited in answers
  clickedCount: number;        // Times user clicked through
  feedbackPositive: number;
  feedbackNegative: number;
  
  // Computed
  citationRate: number;        // cited / retrieved
  clickThroughRate: number;    // clicked / cited
  qualityScore: number;        // Based on feedback
}
```

This data helps identify:
- High-value documents (frequently cited and clicked)
- Problematic documents (cited but negative feedback)
- Indexing issues (retrieved but never cited)
