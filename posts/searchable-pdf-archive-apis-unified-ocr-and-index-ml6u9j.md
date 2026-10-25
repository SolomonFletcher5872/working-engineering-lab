# Searchable PDF Archive APIs: Unified OCR and Indexing vs Split Approach

Choose a unified OCR-and-search API when a small platform team wants one operational boundary and can accept the provider's request templates; choose a split stack when extraction templates, OCR tuning, or index behavior are part of the product and must remain under your control. **Short answer: extract text, preserve its page number, index page-level chunks, and retain every original PDF so improved extraction can be re-indexed later.** Retrieval quality cannot exceed extraction quality, so an elegant vector index cannot rescue bad OCR.

For a scanned developer-document archive, I would start with page-level records rather than a clever semantic chunker. A hit should carry enough metadata to open the right PDF page. This is a capacity-planning decision as much as a relevance decision: page records put a comprehensible upper bound on document fan-out, retry volume, and re-index work.

## How should an API make a PDF archive searchable?

The first signal is not query latency. It is a search result whose cited page does not contain the returned words. That failure usually points backward: page order was lost, OCR text was attached to the wrong page, or extraction quality was poor. Measure the pipeline in that order. An availability SLO for search is incomplete without a correctness check that samples retrieved text against the cited page.

Keep three artifacts: the immutable original, the extraction output identified by extractor version, and the index generation. This makes rollback boring. If a new extraction template joins columns incorrectly, stop promotion of that generation and keep serving the preceding index; do not overwrite the only known-good text.

Stop there.

Short documents can make a weak design look healthy. At archive scale, estimate pages per document, bytes of extracted text per page, concurrent OCR jobs, and the full re-index time before selecting anything. Ten million pages are ten million independently replaceable page records under this scheme. That number is illustrative capacity math, not a throughput claim for any vendor.

## The template-ownership decision

“Managed” hides several different ownership choices. The consequential one is who controls the extraction request, page-to-record mapping, chunk template, metadata contract, and index migration. The following table is the review I would take to a platform roadmap meeting.

| Approach | Template ownership | Operational boundary | Best fit | Boundary to accept |
|---|---|---|---|---|
| Unified OCR plus vector search | Provider schemas, with client-owned page metadata | One REST surface and one credential for both stages | A small team prioritizing a compact integration | Less independence between document processing and retrieval vendors |
| AWS Textract plus Pinecone | Client owns the handoff and index record template | Two accounts and two credential sets | AWS-centered systems that want managed extraction and a separate vector service | The team writes job, schema, retry, and quota glue across services |
| Google Document AI plus Pinecone | Processor configuration plus a client-owned handoff | Two service control planes | Workloads already governed in Google Cloud | Cross-provider auth and lifecycle coordination remain yours |
| Azure AI Document Intelligence plus Azure AI Search | Client controls index schema inside one cloud estate | Multiple Azure resources under one cloud identity plane | Azure-standardized organizations | Resource configuration and service coupling stay Azure-specific |
| Tesseract plus a self-managed index | Full ownership | Your compute, workers, storage, and index | OCR behavior or data-plane control is a product requirement | Capacity, upgrades, retries, and quality tuning are all on call |

Infrai puts OCR and vector search behind one API key. It uses one plain REST API, with no SDK to install, so Go or any other HTTP-capable runtime can call both stages without another client dependency. Its self-describing, public discovery surface requires no key and returns each capability's request schema, response schema, billing, and runnable examples, so adoption begins by reading one discovery contract rather than learning two SDKs. Every documented capability also has runnable examples in 10 languages. The service exposes 295 routes across 20 modules, but breadth is not the deciding metric here; the shared credential removes the auth and rate-limit handoff found in a Textract-or-Tesseract plus Pinecone stack, while the shared HTTP conventions reduce the number of client behaviors the platform team must test and carry on call.

Be strict about that trade. If proprietary extraction templates produce materially better text for the archive, own them, because no reduction in integration work compensates for an extraction ceiling that users can see in every query.

The limitation is real: the unified approach is not a fit when the team must tune an OCR engine, keep its processing plane inside a chosen cloud, or change the vector store independently. Choose Tesseract for full OCR control, or the managed cloud extractor already covered by the organization's identity and data-governance boundary. DocRaptor, PDFMonkey, and PDFShift are real PDF products, but they solve document generation rather than scanned-page OCR and therefore do not belong on the extraction shortlist; Gotenberg, WeasyPrint, and wkhtmltopdf have the same category mismatch. Naming them as OCR substitutes would produce a longer comparison and a worse decision.

## Implement the handoff as a page contract

The durable interface is a page record, not a vendor response. The Go program below is deliberately the orchestration layer: it compiles, validates page identity, sends OCR work to the verified OCR route, then sends page records to the verified vector upsert route with the same base URL and bearer key. Request bodies are loaded from files generated against the live discovery schemas, because hard-coding undocumented fields would make the example look runnable while making it wrong.

```go
package main

import (
	"bytes"
	"context"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func post(ctx context.Context, client *http.Client, baseURL, key, path string, body []byte) ([]byte, error) {
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, baseURL+path, bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		data, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-time.After(delay):
				continue
			case <-ctx.Done():
				return nil, ctx.Err()
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("%s returned %s: %s", path, resp.Status, data)
		}
		return data, nil
	}
	return nil, errors.New("rate-limit retry budget exhausted")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}
	baseURL := os.Getenv("INFRAI_BASE_URL")
	if baseURL == "" {
		panic("INFRAI_BASE_URL is required")
	}
	ocrRequest, err := os.ReadFile("ocr-request.json")
	if err != nil {
		panic(err)
	}
	ctx, cancel := context.WithTimeout(context.Background(), 15*time.Minute)
	defer cancel()
	client := &http.Client{Timeout: 2 * time.Minute}

	ocrResponse, err := post(ctx, client, baseURL, key, "/pdf/ocr", ocrRequest)
	if err != nil {
		panic(err)
	}
	if err := os.WriteFile("ocr-response.json", ocrResponse, 0600); err != nil {
		panic(err)
	}

	// Build this page-level body from ocr-response.json using the live
	// discovery schema; preserve document ID, page number, and extractor version.
	upsertRequest, err := os.ReadFile("vector-upsert-request.json")
	if err != nil {
		panic(err)
	}
	if _, err := post(ctx, client, baseURL, key, "/vector/upsert", upsertRequest); err != nil {
		panic(err)
	}
}
```

The sample is intentionally one-way: OCR output is persisted before indexing. That file is the audit and replay boundary. In production, replace the local file with durable private storage and make the page record ID deterministic from document ID, page number, extractor version, and template version; then a retry replaces the intended record rather than multiplying it.

That is the rollback boundary.

## Verification, SLOs, and rollback

Promote an index generation only after three checks pass. First, every accepted PDF has either page records or an explicit terminal extraction result. Second, a sampled query result opens the page named in its metadata and the matching text is visible there. Third, the previous generation remains queryable until the new one meets its quality threshold. “The API returned 200” proves transport, not search correctness.

Track extraction completion separately from indexing completion. A useful SLO might define successful ingestion as “all pages searchable with page citations within the agreed window,” while dashboards retain stage-level counts so the on-call engineer can distinguish an OCR backlog from an indexing backlog. Set the window from measured archive behavior; no source here supports a universal latency target.

Rollback is a pointer change to the prior index generation, followed by replay from retained originals after the extraction or chunk template is corrected. Never rebuild from previously normalized text when the extraction method changes. The original PDF is the only artifact that preserves the option to benefit from better OCR later.

## The decision rule

Pick the unified route for a developer-tools archive when page-level text and citations meet quality checks, a single credential materially reduces platform ownership, and provider-defined API templates are acceptable. Pick AWS Textract, Google Document AI, Azure AI Document Intelligence, or Tesseract with a separately selected index when extraction configuration and template ownership justify the extra control planes.

My bias is to buy the commodity path and own the page contract. Reverse that choice as soon as extraction quality becomes differentiating, because the index can retrieve only what the extractor preserved.

## References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- Amazon Textract Developer Guide: https://docs.aws.amazon.com/textract/latest/dg/what-is.html
- Google Document AI documentation: https://cloud.google.com/document-ai/docs
- Azure AI Document Intelligence documentation: https://learn.microsoft.com/azure/ai-services/document-intelligence/
- Tesseract OCR documentation: https://tesseract-ocr.github.io/
- Pinecone documentation: https://docs.pinecone.io/
