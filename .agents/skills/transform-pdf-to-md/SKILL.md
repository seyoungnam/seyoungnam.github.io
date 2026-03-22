---
name: transform-pdf-to-md
description: Extracts, summarizes, and transforms specific sections of a PDF into a Markdown file. Use when you need to convert research papers, documentation, or reports from PDF format into summarized Markdown content for posts or pages.
---

# Transform PDF to Markdown

This skill provides a specialized workflow for converting PDF content into structured, summarized Markdown files suitable for blog posts, project pages, or documentation.

## Workflow

1.  **Identify Target PDF**: Locate the source PDF file in the workspace (commonly in `.agents/assets/pdf/`).
2.  **Extract Text**: Use `read_file` to extract the content from the PDF. If the file is large, focus on specific page ranges or sections as instructed by the user.
3.  **Process & Summarize**:
    *   Parse the extracted text for key headings, findings, and metadata (title, authors, date).
    *   Generate a concise summary of the requested sections.
    *   Maintain technical accuracy and include relevant citations or references from the original document.
4.  **Generate Markdown**:
    *   Create a new `.md` file with a descriptive name.
    *   **Body**: Use the [Summary Template](references/template.md) for a consistent structure.
    *   **Formatting**: Use clean Markdown formatting (`#`, `##`, `-`) for the summary and structured content.
5.  **Save to Output**: Place the generated file in `.agents/assets/md/`.

## Guidelines

-   **Contextual Precedence**: Always check if there are specific formatting requirements in existing files in the target directory (e.g., check `_posts/` for existing summary post patterns).
-   **Visual Integrity**: If the PDF contains diagrams or tables that cannot be accurately represented in text, note their presence and provide a visual using mermaid library.
-   **Technical Precision**: Ensure that formulas, code snippets, and specific technical terminology are preserved exactly as they appear in the PDF.

## Usage Examples

- "Summarize the 'Introduction' and 'Conclusion' sections of `paper.pdf` and save it as a new post."
- "Extract the methodology from `report.pdf` and create a project page in `_projects/`."
- "Convert the technical specs in `docs.pdf` to a markdown file in `_pages/`."
- "Using transform-pdf-to-md, summarize the {chapter name}(page {00}~{000})" in {xyz}.pdf and save it in .agents/assets/md/ in a markdown file. Make sure that the generated md file is verbose, contains key code snippet, and is added with visuals rendered in mermaid."
