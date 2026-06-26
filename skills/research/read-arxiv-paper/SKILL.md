---
name: read-arxiv-paper
description: "给定 arxiv URL 时，使用此 skill 读取该 arxiv 论文。"
---

You will be given a URL of an arxiv paper, for example:

https://www.arxiv.org/abs/2601.07372

### Part 1: Normalize the URL

The goal is to fetch the TeX Source of the paper (not the PDF!), the URL always looks like this:

https://www.arxiv.org/src/2601.07372

Notice the /src/ in the url. Once you have the URL:

### Part 2: Download the paper source

Fetch the url to a local .tar.gz file. A good location is `~/.cache/nanochat/knowledge/{arxiv_id}.tar.gz`.

(If the file already exists, there is no need to re-download it).

### Part 3: Unpack the file in that folder

Unpack the contents into `~/.cache/nanochat/knowledge/{arxiv_id}` directory.

### Part 4: Locate the entrypoint

Every latex source usually has an entrypoint, such as `main.tex` or something like that.

### Part 5: Read the paper

Once you've found the entrypoint, Read the contents and then recurse through all other relevant source files to read the paper.

### Part 6: Report

Once you've read the paper, produce a summary of the paper into a markdown file at `./knowledge/summary_{tag}.md`. Notice that 1) use the local knowledge directory here (it's easier for me to open and reference here), not in `~/.cache`, and 2) generate some reasonable `tag` like e.g. `conditional_memory` or whatever seems appropriate given the paper. Probably make sure that the tag doesn't exist yet so you're not overwriting files.

As for the summary itself, remember that you're processing this paper within the context of the nanochat repository, so most often we will be interested in how to apply the paper and its lessons to the nanochat project. Therefore, you should feel free to "reminder yourself" of the related nanochat code by reading the relevant parts, and then explicitly make the connection of how this paper might relate to nanochat or what are things we might be inspired about or try.

### Part 7: Plain-language Explanation

After completing the formal summary, write an additional plain-language explanation section in the same markdown file. This section should:

- Use **simple, everyday language** (avoid academic jargon and heavy math notation)
- Use **concrete analogies or real-world examples** to illustrate core ideas
- Explain **what problem the paper solves** in one sentence
- Explain **how the method works** using a step-by-step analogy (e.g., "imagine you're trying to find your keys in a messy room...")
- Highlight **what makes this approach different** from prior work
- If applicable, include a brief **"what it can do"** section with a simple table of capabilities

The plain-language section should have the header `## 通俗解读` (or `## Plain-Language Explanation` if the user communicates in English), and should be output **only in the conversation reply** — the markdown file should contain only the formal summary from Part 6.
