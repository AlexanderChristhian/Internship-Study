# Markdown Submission Checking — Implementation Guide

> **Date:** 10-Mar-2026
> **Scope:** How to correctly and efficiently implement the checking pipeline for DigiChecker — parsing markdown submissions, handling key answers, ingesting web-based theory content, and structuring the code for readability.

---

## Table of Contents

1. [Submission Format Overview](#1-submission-format-overview)
2. [Markdown Parsing Pipeline](#2-markdown-parsing-pipeline)
3. [Key Answer Upload & Storage](#3-key-answer-upload--storage)
4. [Web Link Ingestion for Theory Content](#4-web-link-ingestion-for-theory-content)
5. [Checking & Grading Pipeline](#5-checking--grading-pipeline)
6. [Image & Screenshot Handling](#6-image--screenshot-handling)
7. [Code Block Extraction & Evaluation](#7-code-block-extraction--evaluation)
8. [Theory Answer Evaluation](#8-theory-answer-evaluation)
9. [Schema & Data Model Changes](#9-schema--data-model-changes)
10. [RAG Service Endpoint Changes](#10-rag-service-endpoint-changes)
11. [Backend Service Layer Changes](#11-backend-service-layer-changes)
12. [Code Structure & Readability](#12-code-structure--readability)
13. [Full Processing Flow Diagram](#13-full-processing-flow-diagram)
14. [Anti-Patterns to Avoid](#14-anti-patterns-to-avoid)
15. [Implementation Checklist](#15-implementation-checklist)

---

## 1. Submission Format Overview

### What a Student Submits

Each student submission is a **Markdown (`.md`) file** that contains a mix of:

| Content Type       | Example in Markdown                              | What It Looks Like                           |
| ------------------ | ------------------------------------------------ | -------------------------------------------- |
| **Theory answers** | Plain paragraphs, bullet points, numbered lists   | Text explanations of concepts                |
| **Code blocks**    | Fenced ` ```python ... ``` ` blocks               | Working code for programming questions       |
| **Screenshots**    | `![output](./screenshots/q3-output.png)`          | Terminal output, GUI results, diagrams       |
| **Headings**       | `## Question 1`, `### Part A`                     | Section separators matching assignment Qs    |

### What the Checker (Lecturer) Provides

1. **Key Answer** — A markdown file with the reference/correct answers for each question.
2. **Theory Web Link** — A URL pointing to a web page (e.g. course notes, textbook chapter) that contains the foundational theory the module covers.

### Core Idea

```
Student Submission (.md)  ──┐
                             ├──→  RAG Comparison  ──→  Grade + Feedback
Key Answer (.md)  ───────────┤
Theory Web Content  ─────────┘
```

The system indexes **both** the key answer and the web theory content into ChromaDB, then evaluates each student submission section against those references.

---

## 2. Markdown Parsing Pipeline

### Why Custom Parsing Matters

Markdown is semi-structured. A naive "dump the whole file as one string" approach loses section boundaries, mixes code with prose, and makes per-question grading impossible. We need to **split by question** and **classify each block**.

### Parsing Strategy

```
Raw .md file
    │
    ▼
┌─────────────────────────┐
│  1. Split by headings   │  → Produces Section[]
│     (## Question N)     │
├─────────────────────────┤
│  2. Classify blocks     │  → Each section has:
│     within each section │     - text_blocks: string[]
│                         │     - code_blocks: CodeBlock[]
│                         │     - image_refs: ImageRef[]
├─────────────────────────┤
│  3. Normalize & clean   │  → Strip excessive whitespace,
│                         │     normalize unicode
└─────────────────────────┘
```

### Data Structures (Python — RAG Service)

```python
# app/models/parsed_submission.py

from dataclasses import dataclass, field


@dataclass
class CodeBlock:
    """A fenced code block extracted from markdown."""
    language: str          # "python", "java", "sql", "" (unspecified)
    code: str              # The raw code content
    line_start: int        # Line number in original file (for feedback)


@dataclass
class ImageRef:
    """An image reference from markdown."""
    alt_text: str          # The alt text: ![alt_text](path)
    path: str              # Relative path or URL
    line_number: int


@dataclass
class SubmissionSection:
    """One logical section (usually one question) from a submission."""
    heading: str           # e.g. "Question 1" or "Part A"
    heading_level: int     # 1 = #, 2 = ##, 3 = ###
    text_blocks: list[str] = field(default_factory=list)
    code_blocks: list[CodeBlock] = field(default_factory=list)
    image_refs: list[ImageRef] = field(default_factory=list)
    raw_content: str = ""  # The full raw markdown of this section


@dataclass
class ParsedSubmission:
    """The complete parsed result of a markdown submission."""
    title: str
    sections: list[SubmissionSection] = field(default_factory=list)
    metadata: dict = field(default_factory=dict)  # filename, student info, etc.
```

### Parsing Implementation

```python
# app/core/markdown_parser.py

import re
from app.models.parsed_submission import (
    CodeBlock,
    ImageRef,
    ParsedSubmission,
    SubmissionSection,
)

# Regex patterns
HEADING_PATTERN = re.compile(r"^(#{1,6})\s+(.+)$", re.MULTILINE)
CODE_FENCE_PATTERN = re.compile(
    r"^```(\w*)\n(.*?)^```",
    re.MULTILINE | re.DOTALL,
)
IMAGE_PATTERN = re.compile(r"!\[([^\]]*)\]\(([^)]+)\)")


def parse_markdown(content: str, filename: str = "") -> ParsedSubmission:
    """Parse a markdown file into structured sections.

    Steps:
      1. Find all heading positions to define section boundaries.
      2. Slice content into sections.
      3. Within each section, extract code blocks, images, and text.
    """
    lines = content.split("\n")
    heading_positions = _find_headings(lines)

    if not heading_positions:
        # No headings found — treat entire content as one section
        section = _parse_section_content("Untitled", 1, content)
        return ParsedSubmission(
            title=filename,
            sections=[section],
            metadata={"filename": filename},
        )

    sections: list[SubmissionSection] = []

    # Content before the first heading (preamble)
    if heading_positions[0][0] > 0:
        preamble = "\n".join(lines[: heading_positions[0][0]])
        if preamble.strip():
            sections.append(_parse_section_content("Preamble", 0, preamble))

    # Each heading-delimited section
    for i, (line_num, level, title) in enumerate(heading_positions):
        end_line = (
            heading_positions[i + 1][0]
            if i + 1 < len(heading_positions)
            else len(lines)
        )
        section_content = "\n".join(lines[line_num:end_line])
        sections.append(_parse_section_content(title, level, section_content))

    return ParsedSubmission(
        title=filename,
        sections=sections,
        metadata={"filename": filename},
    )


def _find_headings(lines: list[str]) -> list[tuple[int, int, str]]:
    """Return list of (line_number, level, title) for each heading."""
    headings = []
    in_code_block = False

    for i, line in enumerate(lines):
        stripped = line.strip()
        if stripped.startswith("```"):
            in_code_block = not in_code_block
            continue
        if in_code_block:
            continue
        match = HEADING_PATTERN.match(stripped)
        if match:
            level = len(match.group(1))
            title = match.group(2).strip()
            headings.append((i, level, title))

    return headings


def _parse_section_content(
    heading: str, level: int, content: str
) -> SubmissionSection:
    """Extract code blocks, images, and text from a section's content."""
    code_blocks = _extract_code_blocks(content)
    image_refs = _extract_images(content)

    # Remove code blocks and images to isolate prose text
    text_only = CODE_FENCE_PATTERN.sub("", content)
    text_only = IMAGE_PATTERN.sub("", text_only)
    text_only = HEADING_PATTERN.sub("", text_only)

    text_blocks = [
        para.strip()
        for para in text_only.split("\n\n")
        if para.strip()
    ]

    return SubmissionSection(
        heading=heading,
        heading_level=level,
        text_blocks=text_blocks,
        code_blocks=code_blocks,
        image_refs=image_refs,
        raw_content=content,
    )


def _extract_code_blocks(content: str) -> list[CodeBlock]:
    """Extract all fenced code blocks from markdown content."""
    blocks = []
    # Track line numbers for each match
    for match in CODE_FENCE_PATTERN.finditer(content):
        language = match.group(1) or ""
        code = match.group(2).strip()
        line_start = content[: match.start()].count("\n") + 1
        blocks.append(CodeBlock(language=language, code=code, line_start=line_start))
    return blocks


def _extract_images(content: str) -> list[ImageRef]:
    """Extract all image references from markdown content."""
    refs = []
    for i, line in enumerate(content.split("\n")):
        for match in IMAGE_PATTERN.finditer(line):
            refs.append(
                ImageRef(
                    alt_text=match.group(1),
                    path=match.group(2),
                    line_number=i + 1,
                )
            )
    return refs
```

### Key Design Decisions

- **Heading-based splitting**: Most assignments use `## Question N` or `### Part A` patterns — this is the natural document structure.
- **Track line numbers**: Essential for generating feedback that says "on line 42 of your submission..."
- **Preserve raw content**: We keep `raw_content` on each section so the LLM can see the full context if needed.
- **Skip headings inside code fences**: A `# comment` inside a Python code block isn't a section heading.

---

## 3. Key Answer Upload & Storage

### Flow

```
Lecturer uploads key_answer.md
         │
         ▼
┌────────────────────────────────┐
│  Backend (Express.js)          │
│  1. Store file metadata in DB  │
│  2. Parse markdown structure   │
│  3. Send to RAG service /index │
└────────────────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│  RAG Service (Python)          │
│  1. Parse into sections        │
│  2. Chunk each section         │
│  3. Embed with BGE-M3          │
│  4. Store in ChromaDB          │
│     collection: "key_answers"  │
└────────────────────────────────┘
```

### ChromaDB Collection Strategy

Use **separate collections** for different content types to keep queries clean:

| Collection         | Contents                       | Metadata Keys                          |
| ------------------ | ------------------------------ | -------------------------------------- |
| `key_answers`      | Chunked key answer sections    | `assignment_id`, `question`, `type`    |
| `theory_content`   | Chunked web page theory        | `assignment_id`, `source_url`, `topic` |
| `submissions`      | Student submission chunks      | `submission_id`, `question`, `type`    |

```python
# app/stores/chroma_store.py  — Updated with collection strategy

COLLECTION_KEY_ANSWERS = "key_answers"
COLLECTION_THEORY = "theory_content"
COLLECTION_SUBMISSIONS = "submissions"

def get_collection(name: str):
    """Get or create a ChromaDB collection by name."""
    client = get_chroma_client()
    return client.get_or_create_collection(
        name=name,
        metadata={"hive:space": "cosine"},
    )
```

### Indexing the Key Answer

```python
# app/services/key_answer_service.py

from app.core.markdown_parser import parse_markdown
from app.core.indexing import chunk_document
from app.core.embeddings import generate_embeddings
from app.stores.chroma_store import get_collection, COLLECTION_KEY_ANSWERS
from app.utils.logger import get_logger

logger = get_logger(__name__)


class KeyAnswerService:
    """Handles parsing, chunking, and indexing of key answer documents."""

    def index(self, assignment_id: int, content: str, filename: str = "") -> int:
        """Index a key answer markdown file into ChromaDB.

        Returns:
            Number of chunks indexed.
        """
        # 1. Parse the markdown into sections
        parsed = parse_markdown(content, filename)

        # 2. Prepare documents for embedding
        documents: list[str] = []
        metadatas: list[dict] = []
        ids: list[str] = []

        for section in parsed.sections:
            # Chunk the text content of each section
            section_text = self._build_section_text(section)
            if not section_text.strip():
                continue

            chunks = chunk_document(section_text)

            for i, chunk in enumerate(chunks):
                doc_id = f"key_{assignment_id}_{section.heading}_{i}"
                documents.append(chunk.text)
                metadatas.append({
                    "assignment_id": assignment_id,
                    "question": section.heading,
                    "type": self._classify_content(section),
                    "chunk_index": i,
                })
                ids.append(doc_id)

        if not documents:
            logger.warning("No content to index", extra={"assignment_id": assignment_id})
            return 0

        # 3. Embed and store
        embeddings = generate_embeddings(documents)
        collection = get_collection(COLLECTION_KEY_ANSWERS)
        collection.upsert(
            ids=ids,
            documents=documents,
            embeddings=embeddings,
            metadatas=metadatas,
        )

        logger.info(
            "Key answer indexed",
            extra={"assignment_id": assignment_id, "chunks": len(documents)},
        )
        return len(documents)

    def delete(self, assignment_id: int) -> None:
        """Remove all key answer chunks for an assignment."""
        collection = get_collection(COLLECTION_KEY_ANSWERS)
        collection.delete(where={"assignment_id": assignment_id})

    def _build_section_text(self, section) -> str:
        """Combine text and code blocks into a single indexable string."""
        parts = []

        for text in section.text_blocks:
            parts.append(text)

        for code_block in section.code_blocks:
            # Prefix code with its language for better embedding context
            lang_label = f"[{code_block.language}] " if code_block.language else ""
            parts.append(f"{lang_label}Code:\n{code_block.code}")

        return "\n\n".join(parts)

    def _classify_content(self, section) -> str:
        """Classify what type of content dominates this section."""
        has_code = len(section.code_blocks) > 0
        has_text = len(section.text_blocks) > 0
        has_images = len(section.image_refs) > 0

        if has_code and has_text:
            return "mixed"
        elif has_code:
            return "code"
        elif has_images:
            return "output"
        else:
            return "theory"
```

### Backend API Endpoint

```typescript
// src/backend/src/controllers/assignment.controller.ts

import type { Request, Response } from "express";
import { keyAnswerService } from "../services/key-answer.service.js";

export async function uploadKeyAnswer(req: Request, res: Response) {
  const assignmentId = Number(req.params.assignmentId);
  const { content, filename } = req.body;

  const result = await keyAnswerService.upload(assignmentId, content, filename);

  res.status(200).json({
    status: "success",
    data: {
      assignmentId,
      chunksIndexed: result.chunksIndexed,
    },
  });
}
```

---

## 4. Web Link Ingestion for Theory Content

### Why Ingest a Web Link?

The lecturer provides a URL (e.g. `https://university.edu/module/cs101/theory`) that contains the foundational theory for the module. This content becomes the **knowledge base** the LLM uses to evaluate whether a student's theory answers are correct and complete.

### Ingestion Pipeline

```
Lecturer provides URL
        │
        ▼
┌───────────────────────────────┐
│  Backend (Express.js)          │
│  1. Validate URL              │
│  2. Forward to RAG service    │
└───────────────────────────────┘
        │
        ▼
┌───────────────────────────────┐
│  RAG Service (Python)          │
│  1. Fetch page content        │
│  2. Extract text (strip HTML) │
│  3. Chunk into passages       │
│  4. Embed with BGE-M3         │
│  5. Store in ChromaDB         │
│     collection: theory_content│
└───────────────────────────────┘
```

### URL Validation & Security (Critical)

**Never blindly fetch any URL.** This is a Server-Side Request Forgery (SSRF) risk.

```python
# app/utils/url_validator.py

from urllib.parse import urlparse
import ipaddress


ALLOWED_SCHEMES = {"http", "https"}

# Block internal/private network ranges
BLOCKED_HOSTS = {"localhost", "127.0.0.1", "0.0.0.0", "::1"}


def validate_url(url: str) -> str:
    """Validate and sanitize URL before fetching.

    Raises ValueError if the URL is invalid or points to a blocked destination.
    """
    parsed = urlparse(url)

    # Scheme check
    if parsed.scheme not in ALLOWED_SCHEMES:
        raise ValueError(f"URL scheme '{parsed.scheme}' is not allowed. Use http or https.")

    # Host check
    hostname = parsed.hostname
    if not hostname:
        raise ValueError("URL must have a valid hostname.")

    if hostname in BLOCKED_HOSTS:
        raise ValueError(f"Access to '{hostname}' is not allowed.")

    # Block private IP ranges
    try:
        ip = ipaddress.ip_address(hostname)
        if ip.is_private or ip.is_loopback or ip.is_reserved:
            raise ValueError("Access to private/reserved IP addresses is not allowed.")
    except ValueError:
        # It's a hostname, not an IP — that's fine
        pass

    return url
```

### Web Content Fetcher

```python
# app/services/web_ingestion_service.py

import httpx
from bs4 import BeautifulSoup

from app.core.indexing import chunk_document
from app.core.embeddings import generate_embeddings
from app.stores.chroma_store import get_collection, COLLECTION_THEORY
from app.utils.url_validator import validate_url
from app.utils.logger import get_logger

logger = get_logger(__name__)

# Maximum content size to prevent abuse (5 MB)
MAX_CONTENT_SIZE = 5 * 1024 * 1024

# Request timeout in seconds
REQUEST_TIMEOUT = 30


class WebIngestionService:
    """Fetches, parses, and indexes web page content into ChromaDB."""

    def ingest(self, assignment_id: int, url: str) -> int:
        """Fetch a web page, extract text, chunk, embed, and store.

        Returns:
            Number of chunks indexed.
        """
        # 1. Validate URL (SSRF protection)
        validated_url = validate_url(url)

        # 2. Fetch the page
        text_content = self._fetch_page(validated_url)

        # 3. Chunk the content
        chunks = chunk_document(text_content)

        if not chunks:
            logger.warning("No content extracted from URL", extra={"url": url})
            return 0

        # 4. Prepare for ChromaDB
        documents = [chunk.text for chunk in chunks]
        ids = [f"theory_{assignment_id}_{i}" for i in range(len(chunks))]
        metadatas = [
            {
                "assignment_id": assignment_id,
                "source_url": validated_url,
                "chunk_index": i,
                "topic": "theory",
            }
            for i in range(len(chunks))
        ]

        # 5. Embed and store
        embeddings = generate_embeddings(documents)
        collection = get_collection(COLLECTION_THEORY)
        collection.upsert(
            ids=ids,
            documents=documents,
            embeddings=embeddings,
            metadatas=metadatas,
        )

        logger.info(
            "Theory content indexed",
            extra={"url": url, "assignment_id": assignment_id, "chunks": len(documents)},
        )
        return len(documents)

    def delete(self, assignment_id: int) -> None:
        """Remove all theory chunks for an assignment."""
        collection = get_collection(COLLECTION_THEORY)
        collection.delete(where={"assignment_id": assignment_id})

    def _fetch_page(self, url: str) -> str:
        """Fetch a web page and extract its text content."""
        with httpx.Client(
            timeout=REQUEST_TIMEOUT,
            follow_redirects=True,
            max_redirects=5,
        ) as client:
            response = client.get(url)
            response.raise_for_status()

            # Size check
            if len(response.content) > MAX_CONTENT_SIZE:
                raise ValueError(
                    f"Page content exceeds maximum size of {MAX_CONTENT_SIZE} bytes."
                )

            return self._extract_text(response.text)

    def _extract_text(self, html: str) -> str:
        """Strip HTML and extract clean text content."""
        soup = BeautifulSoup(html, "html.parser")

        # Remove script, style, nav, footer — irrelevant noise
        for tag in soup(["script", "style", "nav", "footer", "header", "aside"]):
            tag.decompose()

        # Try to find the main content area first
        main = soup.find("main") or soup.find("article") or soup.find("body")
        if main is None:
            return ""

        # Get text with newlines preserved at block boundaries
        text = main.get_text(separator="\n", strip=True)

        # Collapse excessive blank lines
        lines = [line.strip() for line in text.splitlines()]
        return "\n".join(line for line in lines if line)
```

### Dependencies to Add

```toml
# In pyproject.toml [project.dependencies], add:
# httpx>=0.28.0        — async-capable HTTP client
# beautifulsoup4>=4.13  — HTML parsing
```

---

## 5. Checking & Grading Pipeline

### The Full Flow (Per Student Submission)

```
┌─ Lecturer Setup (once per assignment) ─────────────────────────┐
│  1. Upload key_answer.md  → indexed into "key_answers"         │
│  2. Provide theory URL    → fetched & indexed into "theory"    │
└────────────────────────────────────────────────────────────────┘

┌─ Student Submission Check (per student) ──────────────────────┐
│  1. Parse student .md into sections                            │
│  2. For each section:                                          │
│     a. Classify: theory / code / output / mixed                │
│     b. Retrieve relevant key answer chunks (RAG query)         │
│     c. Retrieve relevant theory chunks (RAG query)             │
│     d. Build evaluation prompt with context                    │
│     e. Send to LLM for grading                                 │
│  3. Aggregate section scores into overall grade                │
│  4. Store grade + feedback in database                         │
└────────────────────────────────────────────────────────────────┘
```

### Evaluation Service

```python
# app/services/evaluation_service.py

from app.core.markdown_parser import parse_markdown
from app.core.querying import retrieve_chunks
from app.core.prompts import build_evaluation_prompt
from app.stores.chroma_store import (
    COLLECTION_KEY_ANSWERS,
    COLLECTION_THEORY,
)
from app.models.parsed_submission import SubmissionSection
from app.utils.logger import get_logger

logger = get_logger(__name__)


class SectionResult:
    """Result of evaluating one section of a submission."""

    def __init__(
        self,
        question: str,
        content_type: str,
        score: float,
        feedback: str,
        max_score: float,
    ):
        self.question = question
        self.content_type = content_type
        self.score = score
        self.feedback = feedback
        self.max_score = max_score


class EvaluationResult:
    """Aggregated result of evaluating an entire submission."""

    def __init__(self, sections: list[SectionResult]):
        self.sections = sections
        self.total_score = sum(s.score for s in sections)
        self.max_total = sum(s.max_score for s in sections)
        self.percentage = (
            (self.total_score / self.max_total * 100) if self.max_total > 0 else 0
        )


class EvaluationService:
    """Evaluates a student submission against key answer and theory."""

    def evaluate(
        self,
        assignment_id: int,
        submission_content: str,
        filename: str = "",
    ) -> EvaluationResult:
        """Parse, compare, and grade a student submission.

        Steps:
          1. Parse the submission markdown.
          2. For each section, retrieve relevant reference material.
          3. Build a prompt and call the LLM.
          4. Parse the LLM's response into scores and feedback.
        """
        parsed = parse_markdown(submission_content, filename)
        section_results: list[SectionResult] = []

        for section in parsed.sections:
            if section.heading == "Preamble":
                continue  # Skip preamble (student name, date, etc.)

            result = self._evaluate_section(assignment_id, section)
            section_results.append(result)

        return EvaluationResult(sections=section_results)

    def _evaluate_section(
        self, assignment_id: int, section: SubmissionSection
    ) -> SectionResult:
        """Evaluate a single section against references."""
        content_type = self._classify(section)
        query_text = self._build_query_text(section)

        # Retrieve relevant key answer chunks
        key_chunks = retrieve_chunks(
            query=query_text,
            collection_name=COLLECTION_KEY_ANSWERS,
            where_filter={"assignment_id": assignment_id},
            top_k=5,
        )

        # Retrieve relevant theory chunks
        theory_chunks = retrieve_chunks(
            query=query_text,
            collection_name=COLLECTION_THEORY,
            where_filter={"assignment_id": assignment_id},
            top_k=3,
        )

        # Build the evaluation prompt
        prompt = build_evaluation_prompt(
            student_answer=query_text,
            key_answer_context="\n\n".join(
                chunk["document"] for chunk in key_chunks
            ),
            theory_context="\n\n".join(
                chunk["document"] for chunk in theory_chunks
            ),
            content_type=content_type,
            question_heading=section.heading,
        )

        # Call LLM (placeholder — see section 8 for LLM integration)
        llm_response = self._call_llm(prompt)

        # Parse score and feedback from response
        score, feedback = self._parse_llm_response(llm_response)

        return SectionResult(
            question=section.heading,
            content_type=content_type,
            score=score,
            feedback=feedback,
            max_score=10.0,  # Configurable per assignment
        )

    def _classify(self, section: SubmissionSection) -> str:
        """Classify the dominant content type of a section."""
        has_code = len(section.code_blocks) > 0
        has_text = len(section.text_blocks) > 0
        has_images = len(section.image_refs) > 0

        if has_code and has_text:
            return "mixed"
        elif has_code:
            return "code"
        elif has_images and not has_text:
            return "output"
        else:
            return "theory"

    def _build_query_text(self, section: SubmissionSection) -> str:
        """Build the text representation of a section for RAG querying."""
        parts = [f"Question: {section.heading}"]

        for text in section.text_blocks:
            parts.append(text)

        for code_block in section.code_blocks:
            parts.append(f"```{code_block.language}\n{code_block.code}\n```")

        for img in section.image_refs:
            parts.append(f"[Screenshot: {img.alt_text}]")

        return "\n\n".join(parts)

    def _call_llm(self, prompt: str) -> str:
        """Call the LLM with the evaluation prompt.

        TODO: Integrate with OpenAI / Ollama / LlamaIndex LLM.
        """
        # Placeholder — returns empty until LLM is wired up
        return ""

    def _parse_llm_response(self, response: str) -> tuple[float, str]:
        """Extract score and feedback from LLM response.

        Expected format:
          Score: 7/10
          Feedback: ...
        """
        if not response:
            return 0.0, "Evaluation pending — LLM not configured."

        score = 0.0
        feedback = response

        # Try to find "Score: X/Y" pattern
        import re
        score_match = re.search(r"Score:\s*(\d+(?:\.\d+)?)\s*/\s*\d+", response)
        if score_match:
            score = float(score_match.group(1))

        # Everything after "Feedback:" is the feedback text
        feedback_match = re.search(r"Feedback:\s*(.+)", response, re.DOTALL)
        if feedback_match:
            feedback = feedback_match.group(1).strip()

        return score, feedback
```

---

## 6. Image & Screenshot Handling

### The Challenge

Markdown references images like `![output](./screenshots/q3.png)`, but these are binary files — they can't be directly embedded into a text-based LLM prompt.

### Strategy: Three Approaches (Pick Based on Needs)

| Approach | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **1. Alt-text only** | Use the `![alt text]()` as a description | Simplest, no extra deps | Only works if students write good alt text |
| **2. OCR extraction** | Run OCR on screenshots to extract text | Captures terminal output | Slow, needs Tesseract, noisy for GUIs |
| **3. Vision LLM** | Send image to a multimodal LLM (GPT-4V, Llava) | Best understanding | Most expensive, needs vision-capable model |

### Recommended: Hybrid (Alt-text + Optional OCR)

For a submission checker, most screenshots are **terminal output** or **simple diagrams**. OCR works well for terminal screenshots; alt-text is a good fallback.

```python
# app/core/image_handler.py

import os
from app.utils.logger import get_logger

logger = get_logger(__name__)


def extract_image_context(
    image_refs: list,
    submission_dir: str,
    use_ocr: bool = False,
) -> list[dict]:
    """Extract textual context from image references.

    Returns a list of dicts: { "alt_text": str, "extracted_text": str }
    """
    results = []

    for ref in image_refs:
        context = {"alt_text": ref.alt_text, "extracted_text": ""}

        if use_ocr:
            image_path = os.path.join(submission_dir, ref.path)
            if os.path.exists(image_path):
                context["extracted_text"] = _ocr_image(image_path)

        results.append(context)

    return results


def _ocr_image(image_path: str) -> str:
    """Run OCR on an image file to extract text.

    Uses pytesseract if available; returns empty string otherwise.
    """
    try:
        from PIL import Image
        import pytesseract

        image = Image.open(image_path)
        text = pytesseract.image_to_string(image)
        return text.strip()
    except ImportError:
        logger.warning("pytesseract not installed — skipping OCR")
        return ""
    except Exception as e:
        logger.error("OCR failed", extra={"path": image_path, "error": str(e)})
        return ""
```

### Incorporating Image Context into Evaluation

When building the evaluation prompt (Section 5), include any extracted image text:

```python
# In the evaluation prompt builder:
if section.image_refs:
    image_contexts = extract_image_context(
        section.image_refs,
        submission_dir=upload_dir,
        use_ocr=settings.enable_ocr,
    )
    for ctx in image_contexts:
        if ctx["extracted_text"]:
            prompt_parts.append(
                f"Screenshot ({ctx['alt_text']}): {ctx['extracted_text']}"
            )
        else:
            prompt_parts.append(
                f"Screenshot ({ctx['alt_text']}): [image not analyzed]"
            )
```

---

## 7. Code Block Extraction & Evaluation

### What to Check in Code Blocks

| Check | Method | Description |
|-------|--------|-------------|
| **Correctness** | Semantic comparison with key answer | Does the code logic match the expected answer? |
| **Completeness** | Key concepts coverage | Does it use the required techniques/libraries? |
| **Style** | Optional linting | Follows coding conventions (not usually graded) |

### Embedding Code for Comparison

Code blocks should be embedded **with language context** so the embedding model can distinguish Python from Java:

```python
def prepare_code_for_embedding(code_block: CodeBlock) -> str:
    """Format a code block for embedding with language context."""
    lang = code_block.language or "unknown"
    return f"Programming language: {lang}\n\nCode:\n{code_block.code}"
```

### Code-Specific Evaluation Prompt

```python
CODE_EVALUATION_TEMPLATE = """You are evaluating a student's code submission.

**Question:** {question}

**Expected Answer (Key):**
```{key_language}
{key_code}
```

**Student's Answer:**
```{student_language}
{student_code}
```

**Theory Context:**
{theory_context}

Evaluate the student's code on:
1. **Correctness** (does it produce the right output / solve the problem?)
2. **Completeness** (does it cover all required aspects?)
3. **Understanding** (does the code show the student understands the concept?)

Provide:
- Score: X/10
- Feedback: Specific, constructive feedback referencing exact lines or patterns.
"""
```

### Important: Do NOT Execute Student Code

Never run student-submitted code. This is a **major security risk** (arbitrary code execution). All evaluation should be done through:
1. Semantic comparison (embeddings)
2. LLM-based analysis (reading the code, not running it)

---

## 8. Theory Answer Evaluation

### How Theory Checking Works

For theory/text answers, the pipeline is:

```
Student's text answer
        │
        ├─→ Embed & query against key_answers  → relevant reference chunks
        │
        ├─→ Embed & query against theory_content → relevant theory chunks
        │
        └─→ Build prompt:
              "Given the key answer and theory, evaluate this student answer"
                      │
                      ▼
                  LLM grades it
```

### Theory-Specific Evaluation Prompt

```python
THEORY_EVALUATION_TEMPLATE = """You are evaluating a student's theory answer for an academic assignment.

**Question:** {question}

**Reference Answer (Key):**
{key_answer}

**Background Theory:**
{theory_context}

**Student's Answer:**
{student_answer}

Evaluate the student's answer on:
1. **Accuracy** — Are the claims factually correct based on the theory?
2. **Completeness** — Does it cover all key points from the reference answer?
3. **Depth** — Does it show understanding beyond surface-level repetition?
4. **Clarity** — Is the explanation clear and well-structured?

Respond in EXACTLY this format:
Score: X/10
Feedback: <your detailed feedback here>
"""
```

### Handling Mixed Sections

Some questions have both theory text AND code. For these, evaluate both parts and combine:

```python
def evaluate_mixed_section(section, key_chunks, theory_chunks):
    """Evaluate a section that contains both theory and code."""
    theory_score, theory_feedback = evaluate_theory(
        section.text_blocks, key_chunks, theory_chunks
    )
    code_score, code_feedback = evaluate_code(
        section.code_blocks, key_chunks, theory_chunks
    )

    # Weight: 40% theory, 60% code (configurable)
    combined_score = (theory_score * 0.4) + (code_score * 0.6)
    combined_feedback = (
        f"**Theory ({theory_score}/10):** {theory_feedback}\n\n"
        f"**Code ({code_score}/10):** {code_feedback}"
    )

    return combined_score, combined_feedback
```

---

## 9. Schema & Data Model Changes

### New Prisma Models Needed

The current schema needs additions to support key answers and theory sources:

```prisma
// ── Key Answers ───────────────────────────────────────────

model KeyAnswer {
  id           Int        @id @default(autoincrement())
  assignmentId Int        @unique       // One key answer per assignment
  assignment   Assignment @relation(fields: [assignmentId], references: [id])
  content      String     @db.Text      // Raw markdown content
  fileName     String?
  chromaIndexed Boolean   @default(false)
  createdAt    DateTime   @default(now())
  updatedAt    DateTime   @updatedAt
}

// ── Theory Sources ────────────────────────────────────────

model TheorySource {
  id           Int        @id @default(autoincrement())
  assignmentId Int
  assignment   Assignment @relation(fields: [assignmentId], references: [id])
  url          String                   // The web link provided by lecturer
  title        String?                  // Extracted page title
  chromaIndexed Boolean   @default(false)
  createdAt    DateTime   @default(now())

  @@index([assignmentId])
}
```

### Updated Assignment Model

```prisma
model Assignment {
  id        Int      @id @default(autoincrement())
  title     String
  courseId   Int
  course    Course   @relation(fields: [courseId], references: [id])
  dueDate   DateTime?
  createdAt DateTime @default(now())

  submissions   Submission[]
  keyAnswer     KeyAnswer?       // ← Add this relation
  theorySources TheorySource[]   // ← Add this relation
}
```

### Updated Submission Model

Add a JSON field to store per-section grades:

```prisma
model Submission {
  // ... existing fields ...
  sectionGrades Json?   // Stores: [{ question, type, score, maxScore, feedback }]
}
```

---

## 10. RAG Service Endpoint Changes

### New Endpoints Needed

```
POST   /key-answer/index        — Index a key answer for an assignment
DELETE /key-answer/{id}          — Remove key answer from index
POST   /theory/ingest            — Fetch & index a web URL
DELETE /theory/{assignment_id}   — Remove theory content
POST   /evaluate                 — Evaluate a student submission
```

### Updated Pydantic Schemas

```python
# app/api/schemas.py  — Add these models

class KeyAnswerIndexRequest(BaseModel):
    assignment_id: int
    content: str
    filename: str = ""


class TheoryIngestRequest(BaseModel):
    assignment_id: int
    url: str


class EvaluateRequest(BaseModel):
    assignment_id: int
    submission_id: int
    content: str
    filename: str = ""


class SectionFeedback(BaseModel):
    question: str
    content_type: str       # "theory", "code", "mixed", "output"
    score: float
    max_score: float
    feedback: str


class EvaluateResponse(BaseModel):
    submission_id: int
    sections: list[SectionFeedback]
    total_score: float
    max_total: float
    percentage: float
```

### Route Registration

```python
# app/api/routes.py  — Add new routes

@router.post("/key-answer/index", response_model=IndexResponse)
async def index_key_answer(request: KeyAnswerIndexRequest):
    """Index a key answer document for an assignment."""
    chunks = key_answer_service.index(
        request.assignment_id, request.content, request.filename,
    )
    return IndexResponse(status="indexed", chunks=chunks)


@router.post("/theory/ingest", response_model=IndexResponse)
async def ingest_theory(request: TheoryIngestRequest):
    """Fetch and index theory content from a web URL."""
    chunks = web_ingestion_service.ingest(
        request.assignment_id, request.url,
    )
    return IndexResponse(status="indexed", chunks=chunks)


@router.post("/evaluate", response_model=EvaluateResponse)
async def evaluate_submission(request: EvaluateRequest):
    """Evaluate a student submission against key answer and theory."""
    result = evaluation_service.evaluate(
        request.assignment_id, request.content, request.filename,
    )
    return EvaluateResponse(
        submission_id=request.submission_id,
        sections=[
            SectionFeedback(
                question=s.question,
                content_type=s.content_type,
                score=s.score,
                max_score=s.max_score,
                feedback=s.feedback,
            )
            for s in result.sections
        ],
        total_score=result.total_score,
        max_total=result.max_total,
        percentage=result.percentage,
    )
```

---

## 11. Backend Service Layer Changes

### Key Answer Service (Express.js side)

```typescript
// src/backend/src/services/key-answer.service.ts

import { prisma } from "../config/database.js";
import { ragClient } from "./rag.service.js";
import { logger } from "../utils/logger.js";

export const keyAnswerService = {
  async upload(assignmentId: number, content: string, filename?: string) {
    // 1. Store in database
    const keyAnswer = await prisma.keyAnswer.upsert({
      where: { assignmentId },
      create: {
        assignmentId,
        content,
        fileName: filename ?? null,
        chromaIndexed: false,
      },
      update: {
        content,
        fileName: filename ?? null,
        chromaIndexed: false,
      },
    });

    // 2. Index in RAG service
    const result = await ragClient.indexKeyAnswer(assignmentId, content, filename);

    // 3. Mark as indexed
    await prisma.keyAnswer.update({
      where: { id: keyAnswer.id },
      data: { chromaIndexed: true },
    });

    logger.info({ assignmentId, chunks: result.chunks }, "Key answer indexed");

    return { keyAnswerId: keyAnswer.id, chunksIndexed: result.chunks };
  },
};
```

### Submission Check Service (Express.js side)

```typescript
// src/backend/src/services/check.service.ts

import { prisma } from "../config/database.js";
import { ragClient } from "./rag.service.js";
import { logger } from "../utils/logger.js";

export const checkService = {
  async checkSubmission(submissionId: number) {
    // 1. Get submission with assignment info
    const submission = await prisma.submission.findUniqueOrThrow({
      where: { id: submissionId },
      include: { assignment: true },
    });

    if (!submission.assignmentId) {
      throw new Error("Submission is not linked to an assignment.");
    }

    // 2. Update status to PROCESSING
    await prisma.submission.update({
      where: { id: submissionId },
      data: { status: "PROCESSING" },
    });

    try {
      // 3. Call RAG service to evaluate
      const result = await ragClient.evaluate(
        submission.assignmentId,
        submissionId,
        submission.content,
        submission.fileName ?? undefined,
      );

      // 4. Store grade
      await prisma.grade.create({
        data: {
          submissionId,
          score: result.percentage,
          feedback: result.sections
            .map(
              (s) => `**${s.question}** (${s.score}/${s.max_score}): ${s.feedback}`
            )
            .join("\n\n"),
        },
      });

      // 5. Store per-section grades as JSON
      await prisma.submission.update({
        where: { id: submissionId },
        data: {
          status: "CHECKED",
          sectionGrades: result.sections,
        },
      });

      logger.info(
        { submissionId, score: result.percentage },
        "Submission checked",
      );

      return result;
    } catch (err) {
      await prisma.submission.update({
        where: { id: submissionId },
        data: { status: "FAILED" },
      });
      throw err;
    }
  },
};
```

### Updated RAG Client

```typescript
// Add to src/backend/src/services/rag.service.ts

export const ragClient = {
  // ... existing methods ...

  async indexKeyAnswer(assignmentId: number, content: string, filename?: string) {
    const response = await fetch(`${config.RAG_SERVICE_URL}/key-answer/index`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        assignment_id: assignmentId,
        content,
        filename: filename ?? "",
      }),
    });
    return response.json() as Promise<{ status: string; chunks: number }>;
  },

  async ingestTheory(assignmentId: number, url: string) {
    const response = await fetch(`${config.RAG_SERVICE_URL}/theory/ingest`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ assignment_id: assignmentId, url }),
    });
    return response.json() as Promise<{ status: string; chunks: number }>;
  },

  async evaluate(
    assignmentId: number,
    submissionId: number,
    content: string,
    filename?: string,
  ) {
    const response = await fetch(`${config.RAG_SERVICE_URL}/evaluate`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        assignment_id: assignmentId,
        submission_id: submissionId,
        content,
        filename: filename ?? "",
      }),
    });
    return response.json() as Promise<{
      submission_id: number;
      sections: Array<{
        question: string;
        content_type: string;
        score: number;
        max_score: number;
        feedback: string;
      }>;
      total_score: number;
      max_total: number;
      percentage: number;
    }>;
  },
};
```

---

## 12. Code Structure & Readability

### File Organization in RAG Service

After all changes, the RAG service should look like:

```
rag-service/
├── app/
│   ├── config.py                    # Settings (pydantic-settings)
│   ├── main.py                      # FastAPI entrypoint
│   ├── api/
│   │   ├── routes.py                # All route handlers
│   │   └── schemas.py               # Pydantic request/response models
│   ├── core/
│   │   ├── embeddings.py            # BGE-M3 embedding functions
│   │   ├── indexing.py              # LlamaIndex chunking
│   │   ├── markdown_parser.py       # ← NEW: Markdown → sections
│   │   ├── image_handler.py         # ← NEW: OCR / image context
│   │   ├── querying.py              # ChromaDB retrieval
│   │   └── prompts.py               # ← UPDATED: All prompt templates
│   ├── models/
│   │   └── parsed_submission.py     # ← NEW: Data classes
│   ├── services/
│   │   ├── document_service.py      # Text cleaning
│   │   ├── evaluation_service.py    # ← NEW: Main grading orchestrator
│   │   ├── index_service.py         # Submission indexing
│   │   ├── key_answer_service.py    # ← NEW: Key answer indexing
│   │   ├── query_service.py         # RAG query execution
│   │   ├── similarity_service.py    # Cross-submission similarity
│   │   └── web_ingestion_service.py # ← NEW: Web scraping + indexing
│   ├── stores/
│   │   ├── chroma_store.py          # ← UPDATED: Multi-collection
│   │   └── vector_store.py          # LlamaIndex wrappers
│   └── utils/
│       ├── logger.py                # Structured logging
│       ├── text_processing.py       # Text normalization
│       └── url_validator.py         # ← NEW: SSRF protection
├── data/
│   ├── chroma_db/
│   └── uploads/
├── tests/
│   ├── conftest.py
│   ├── test_markdown_parser.py      # ← NEW
│   ├── test_evaluation.py           # ← NEW
│   └── test_routes.py
└── pyproject.toml
```

### Naming Conventions

| Category | Convention | Example |
|----------|-----------|---------|
| Files | `snake_case.py`, `kebab-case.ts` | `markdown_parser.py`, `key-answer.service.ts` |
| Classes | `PascalCase` | `EvaluationService`, `ParsedSubmission` |
| Functions | `snake_case` (Python), `camelCase` (TS) | `parse_markdown()`, `checkSubmission()` |
| Constants | `UPPER_SNAKE_CASE` | `COLLECTION_KEY_ANSWERS`, `MAX_CONTENT_SIZE` |
| Variables | `snake_case` (Python), `camelCase` (TS) | `section_results`, `chunksIndexed` |

### Readability Principles

1. **One responsibility per file.** `markdown_parser.py` only parses markdown. `evaluation_service.py` only orchestrates evaluation. Don't mix concerns.

2. **Small, focused functions.** Each function does one thing and is ≤ 30 lines. If a function is getting long, extract a helper — but only if it has a clear name and purpose.

3. **Clear data flow.** Always know where data comes from and where it goes:
   ```
   Controller → Service → Core logic → Store
   ```
   Never let a controller call a store directly, or a service import from a controller.

4. **Type everything.**
   - Python: use dataclasses or Pydantic models, not raw dicts.
   - TypeScript: define interfaces for all API responses.

5. **Explicit over implicit.**
   ```python
   # Bad: What does this dict contain?
   result = {"s": 7, "f": "good work"}

   # Good: Self-documenting structure
   result = SectionResult(
       question="Question 1",
       score=7,
       feedback="Good work",
   )
   ```

6. **Group imports by category.** Standard lib → third-party → local, separated by blank lines.

7. **Consistent error handling pattern.**
   ```python
   # In routes: catch and convert to HTTP errors
   try:
       result = service.do_thing(request.data)
       return ResponseModel(...)
   except ValueError as e:
       raise HTTPException(status_code=400, detail=str(e))
   except Exception as e:
       logger.error("Operation failed", extra={"error": str(e)})
       raise HTTPException(status_code=500, detail="Internal error")
   ```

---

## 13. Full Processing Flow Diagram

```
                    ┌─────────────┐
                    │   Lecturer   │
                    └──────┬──────┘
                           │
                ┌──────────┴──────────┐
                │                     │
                ▼                     ▼
       Upload key_answer.md    Provide theory URL
                │                     │
                ▼                     ▼
    ┌───────────────────┐  ┌────────────────────┐
    │ Express.js Backend │  │ Express.js Backend  │
    │ POST /assignments/ │  │ POST /assignments/  │
    │   :id/key-answer   │  │   :id/theory-link   │
    └────────┬──────────┘  └─────────┬──────────┘
             │                       │
             ▼                       ▼
    ┌───────────────────┐  ┌────────────────────┐
    │  Store in Prisma   │  │  Store URL in       │
    │  (KeyAnswer model) │  │  Prisma             │
    └────────┬──────────┘  │  (TheorySource)      │
             │              └─────────┬──────────┘
             ▼                        ▼
    ┌───────────────────┐  ┌────────────────────┐
    │  RAG: Parse .md    │  │  RAG: Fetch URL     │
    │  Chunk sections    │  │  Extract text        │
    │  Embed (BGE-M3)    │  │  Chunk & embed       │
    │  → ChromaDB        │  │  → ChromaDB          │
    │  "key_answers"     │  │  "theory_content"    │
    └───────────────────┘  └────────────────────┘


                    ┌─────────────┐
                    │   Student    │
                    └──────┬──────┘
                           │
                    Upload submission.md
                           │
                           ▼
              ┌──────────────────────┐
              │  Express.js Backend   │
              │  POST /submissions    │
              └───────────┬──────────┘
                          │
                          ▼
              ┌──────────────────────┐
              │  Store in Prisma      │
              │  status = PENDING     │
              └───────────┬──────────┘
                          │
                 Lecturer clicks "Check"
                          │
                          ▼
              ┌──────────────────────┐
              │  POST /submissions/   │
              │    :id/check          │
              │  status → PROCESSING  │
              └───────────┬──────────┘
                          │
                          ▼
              ┌──────────────────────┐
              │  RAG: /evaluate       │
              │                       │
              │  1. Parse student .md │
              │  2. For each section: │
              │     - Classify type   │
              │     - Retrieve key    │
              │       answer chunks   │
              │     - Retrieve theory │
              │       chunks          │
              │     - Build prompt    │
              │     - Call LLM        │
              │  3. Aggregate scores  │
              └───────────┬──────────┘
                          │
                          ▼
              ┌──────────────────────┐
              │  Store results:       │
              │  - Grade record       │
              │  - sectionGrades JSON │
              │  status → CHECKED     │
              └──────────────────────┘
```

---

## 14. Anti-Patterns to Avoid

### 1. Monolithic Evaluation Prompt

**Don't** dump the entire key answer + entire submission + entire theory into one giant prompt.

```python
# BAD — will hit token limits and reduce accuracy
prompt = f"""
Key Answer: {entire_key_answer_file}
Theory: {entire_theory_content}
Student: {entire_student_submission}
Grade everything.
"""
```

**Do** evaluate per-section with targeted retrieval:

```python
# GOOD — focused, relevant context per question
for section in parsed.sections:
    relevant_key = retrieve_chunks(section.heading, "key_answers", top_k=5)
    relevant_theory = retrieve_chunks(section.heading, "theory_content", top_k=3)
    grade_section(section, relevant_key, relevant_theory)
```

### 2. Running Student Code

**Never** execute code from submissions. It's arbitrary code from an untrusted source.

### 3. Trusting OCR Blindly

OCR output is noisy. Always treat it as supplementary context, not ground truth.

### 4. One Collection for Everything

**Don't** mix key answers, theory, and submissions in one ChromaDB collection. Metadata filtering works, but separate collections make queries cleaner and faster.

### 5. Fetching URLs Without Validation

Always validate URLs before fetching. Block private IPs, internal hosts, and non-HTTP schemes. See Section 4 for the `validate_url()` implementation.

### 6. Hardcoding Weights and Thresholds

```python
# BAD — hardcoded, can't adjust per assignment
combined_score = theory_score * 0.4 + code_score * 0.6

# GOOD — configurable per assignment
weights = assignment.grading_weights or {"theory": 0.4, "code": 0.6}
combined_score = theory_score * weights["theory"] + code_score * weights["code"]
```

---

## 15. Implementation Checklist

### Phase 1: Core Parsing (No LLM Needed)

- [ ] Create `app/models/parsed_submission.py` — Data classes
- [ ] Create `app/core/markdown_parser.py` — Parse markdown into sections
- [ ] Write tests for markdown parser (headings, code blocks, images, edge cases)
- [ ] Create `app/utils/url_validator.py` — SSRF-safe URL validation

### Phase 2: Key Answer & Theory Ingestion

- [ ] Update `app/stores/chroma_store.py` — Multi-collection support
- [ ] Create `app/services/key_answer_service.py` — Index key answers
- [ ] Create `app/services/web_ingestion_service.py` — Fetch & index web content
- [ ] Add `httpx` and `beautifulsoup4` to dependencies
- [ ] Add new API endpoints in routes
- [ ] Add new Pydantic schemas

### Phase 3: Database Schema

- [ ] Add `KeyAnswer` model to Prisma schema
- [ ] Add `TheorySource` model to Prisma schema
- [ ] Add `sectionGrades` field to `Submission` model
- [ ] Update `Assignment` relations
- [ ] Run migration

### Phase 4: Backend Integration

- [ ] Create `key-answer.service.ts` — Upload + index orchestration
- [ ] Create `check.service.ts` — Full check workflow
- [ ] Update `rag.service.ts` — New RAG client methods
- [ ] Add new routes for key answer upload and theory link

### Phase 5: Evaluation Pipeline

- [ ] Create `app/services/evaluation_service.py` — Section-by-section grading
- [ ] Update `app/core/prompts.py` — Evaluation prompt templates
- [ ] Create `app/core/image_handler.py` — Optional OCR support
- [ ] Update `app/core/querying.py` — Support multi-collection queries
- [ ] Add `/evaluate` endpoint

### Phase 6: LLM Integration

- [ ] Choose LLM provider (OpenAI API / Ollama local)
- [ ] Wire LLM calls into `EvaluationService._call_llm()`
- [ ] Add response parsing and validation
- [ ] Add retry logic for LLM failures
- [ ] Test end-to-end pipeline

### Phase 7: Testing & Polish

- [ ] Integration tests for full check pipeline
- [ ] Edge cases: empty submissions, missing key answer, broken images
- [ ] Performance: batch embedding calls, async where possible
- [ ] Rate limiting on theory URL fetching
