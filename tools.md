# Tools

- Arxiv Search
```
import arxiv
import time
from typing import List, Dict, Any


def arxiv_search(query: str, max_results: int = 5, retries: int = 3) -> Dict[str, Any]:
    """
    Search for research papers using the ArXiv API.

    Args:
        query (str): The search query.
        max_results (int): Maximum number of results to return.
        retries (int): Number of retry attempts on failure.

    Returns:
        dict: A dictionary containing the 'status' and the resulting 'data' or 'message'.
    """
    print(f"[Tool: arxiv_search] Fetching papers for: '{query}'...")

    for attempt in range(retries):
        try:
            client = arxiv.Client(
                page_size=max_results,
                delay_seconds=3.0,
            )

            search = arxiv.Search(
                query=query,
                max_results=max_results,
                sort_by=arxiv.SortCriterion.Relevance,
            )

            results: List[Dict[str, Any]] = []

            for paper in client.results(search):
                results.append({
                    "title": paper.title,
                    "authors": [author.name for author in paper.authors],
                    "published": str(paper.published.date()),
                    "pdf_url": paper.pdf_url,
                    "arxiv_id": paper.get_short_id(),
                    "abstract": paper.summary.replace("\n", " "),
                })

            if not results:
                return {
                    "status": "success",
                    "message": "No papers found for this query on ArXiv.",
                    "data": [],
                }

            return {
                "status": "success",
                "data": results,
            }

        except Exception as e:
            wait_time = 2**attempt
            print(f"Attempt {attempt + 1} failed: {e}")

            if attempt < retries - 1:
                print(f"Retrying in {wait_time} seconds...")
                time.sleep(wait_time)
            else:
                return {
                    "status": "error",
                    "message": f"Error executing ArXiv search after {retries} attempts: {str(e)}",
                }
```

- Pdf Parser
  ```
  import os
  import tempfile

  import fitz 
  import requests
  
  
  def pdf_extract(url: str) -> str:
      """
      Downloads and extracts text from a given PDF URL or local file path.

    Args:
        url (str): The URL or local path of the PDF document.

    Returns:
        str: Extracted text from the PDF.
    """
    print(f"[Tool: pdf_extract] Fetching PDF from: '{url}'...")
    try:
        if os.path.exists(url):
            temp_path = url
            to_delete = False
        else:
            response = requests.get(url, timeout=10)
            response.raise_for_status()

            with tempfile.NamedTemporaryFile(delete=False, suffix=".pdf") as temp_pdf:
                temp_pdf.write(response.content)
                temp_path = temp_pdf.name
            to_delete = True

        text_content = []
        with fitz.open(temp_path) as doc:
            for page in doc:
                text_content.append(page.get_text())

        if to_delete:
            os.remove(temp_path)
        return "\n".join(text_content)[:15000]
    except Exception as e:
        return f"Error extracting PDF: {str(e)}"
