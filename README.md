 MDLocal PixelRAG — Chat With Your Own PDFs

A small, self-contained Jupyter notebook that lets you ask questions over your own PDFs and reports using a PixelRAG-style pipeline — no text extraction, no chunking, no OCR. Every page is rendered as an image and read visually, tables, charts, and layout included.

Built on top of the ideas behind PixelRAG (Berkeley SkyLab / BAIR / Berkeley NLP), but running fully locally over your own private documents instead of the hosted Wikipedia index.

Why

Traditional RAG pipelines parse PDFs into text, chunk that text, and embed the chunks. That process is blind to anything visual: tables get flattened, charts disappear, and multi-column layouts turn into scrambled text. This project skips text extraction entirely. Pages are rendered as images and handed straight to a vision-language model, the same way a person would actually read them.

Your PDFs  ->  Render pages to images  ->  Embed images (CLIP)  ->  Local vector index
                                                                          |
                                                                          v
Your Question  ->  Embed query (CLIP text encoder)  ->  Retrieve top-k page images
                                                                          |
                                                                          v
                                              Claude (vision model) reads the images -> Answer

Features


No text extraction — pages are rasterized and searched as pixels, not parsed as text
Local, private indexing — rendering, embedding, and retrieval all run on your machine; only the final top-k images are sent to the Claude API to generate the answer
Automatic tiling — unusually tall pages are sliced into fixed-height tiles, the same approach PixelRAG uses for long web pages
Source-cited answers — every answer references the source file and page number it came from
No vector database required — brute-force cosine similarity in numpy is enough for a few thousand pages; swap in FAISS if your collection grows past that


Requirements


Python 3.10+
An ANTHROPIC_API_KEY environment variable (used only for the final answer-generation step)
~5 minutes on first run to download CLIP model weights


Installation

bashgit clone <this-repo-url>
cd local-pixelrag
pip install pymupdf pillow numpy torch open_clip_torch anthropic tqdm matplotlib

Usage


Open pixelrag_local_pdf_project.ipynb in Jupyter.
Drop your PDF files into the my_pdfs/ folder (created automatically on first run).
Run the notebook top to bottom:

Section 3 renders every PDF page to an image tile.
Section 4 embeds every tile with CLIP and saves a local index.
Section 5–6 implement retrieval and the vision-model reading step.
Section 8 is where you actually ask a question:





python     answer, retrieved_tiles = pixel_rag_query(
         "What was the total revenue reported in the Q3 table?",
         k=5
     )
     print(answer)


Re-run sections 3–4 any time you add or change PDFs.


How it works

StepWhat happensToolRenderEach PDF page is rasterized to a PNG imagePyMuPDFTilePages taller than a threshold are sliced into fixed-height tilesPillowEmbedEvery tile is embedded into a shared image/text vector spaceCLIP (open_clip)IndexEmbeddings + metadata (source file, page, tile index) are saved to diskNumPy / JSONRetrieveThe text query is embedded in the same space; top-k tiles are found by cosine similarityNumPyReadRetrieved page images are sent directly to a vision-capable Claude model, which answers using layout and visual content, not extracted textAnthropic API

Configuration

All the knobs live in the config cell near the top of the notebook:

VariableDefaultPurposePDF_DIR./my_pdfsFolder containing your source PDFsRENDER_DPI150Rasterization resolution — higher is sharper but slower and more expensive per queryMAX_TILE_HEIGHT_PX1600Height threshold before a page gets sliced into multiple tiles

Known limitations


Fixed-height tiling has no content awareness. A table or paragraph that straddles a tile boundary can get cut in half. For normal single-page-sized PDFs this rarely matters, since one page usually fits in one tile.
Cost scales with k and RENDER_DPI. Each query sends k full-resolution images to the API (~1,500–1,600 tokens per image at 150 DPI). Tune both down if cost matters more than recall.
Brute-force search doesn't scale indefinitely. Fine for a personal document collection; for tens of thousands of pages, swap the retrieval step for FAISS or another approximate nearest-neighbor index.




+2+2=1
