---
name: Library/doi-paper/DOI Paper
tags: meta/library
---

# DOI Paper Import

A Space Lua library for creating research paper notes from a DOI. Supports papers registered with CrossRef (ACM Digital Library, IEEE Xplore, and most other publishers) as well as arXiv preprints.

## Usage

Run the **Paper: Import from DOI** command from the command palette. Enter a DOI (e.g. `10.1145/3582016.3582063`) or an arXiv ID (e.g. `2301.13808`), and a new page will be created under `Papers/` with frontmatter populated from the paper's metadata.

## Supported Identifiers

| Input Format | Example                       | Resolution Method |
| ------------ | ----------------------------- | ----------------- |
| DOI (bare)   | `10.1145/3582016.3582063`     | CrossRef REST API |
| DOI (URL)    | `https://doi.org/10.1145/...` | CrossRef REST API |
| arXiv ID     | `2301.13808`                  | arXiv Atom API    |
| arXiv DOI    | `10.48550/arXiv.2301.13808`   | arXiv Atom API    |

## Generated Frontmatter

```yaml
title: "Paper Title Here"
authors:
  - "First Author"
  - "Second Author"
doi: "10.1145/..."
url: "https://doi.org/10.1145/..."
venue: "Conference or Journal Name"
publisher: "ACM"
date: "2023-06-17"
year: 2023
type: paper
tags:
  - paper
  - unread
abstract: "The paper abstract..."
```

## Configuration

You can customize the page path prefix and default tags in your `CONFIG` page:

```space-lua
config.set {
  doiPaper = {
    prefix = "papers",       -- page path prefix (default: "Papers")
    tags = {"paper", "unread"} -- default tags (default: {"paper", "unread"})
  }
}
```

## Implementation

```space-lua
-- priority: 5

doi_paper = doi_paper or {}

-- Sanitize a title for use as a page name
function doi_paper.sanitize_page_name(title)
  -- Remove characters that are problematic in page names
  local clean = title
  clean = string.gsub(clean, "[/\\#%[%]|{}]", "")
  clean = string.gsub(clean, "%s+", " ")
  clean = string.trim(clean)
  -- Truncate to reasonable length
  if #clean > 120 then
    clean = string.sub(clean, 1, 120)
  end
  return clean
end

-- Detect identifier type from user input
function doi_paper.parse_identifier(input)
  input = string.trim(input)

  -- Strip URL prefixes
  input = string.gsub(input, "^https?://doi%.org/", "")
  input = string.gsub(input, "^https?://dx%.doi%.org/", "")
  input = string.gsub(input, "^https?://arxiv%.org/abs/", "")

  -- Check if it's an arXiv DOI (10.48550/arXiv.XXXX.XXXXX)
  local arxiv_from_doi = string.match(input, "^10%.48550/arXiv%.(.+)$")
  if arxiv_from_doi then
    return { type = "arxiv", id = arxiv_from_doi }
  end

  -- Check if it's a bare arXiv ID (YYMM.NNNNN or old format like hep-th/9901001)
  if string.match(input, "^%d%d%d%d%.%d%d%d%d%d?v?%d*$") then
    -- Strip version suffix if present
    local base = string.gsub(input, "v%d+$", "")
    return { type = "arxiv", id = base }
  end
  if string.match(input, "^[a-z%-]+/%d+v?%d*$") then
    local base = string.gsub(input, "v%d+$", "")
    return { type = "arxiv", id = base }
  end

  -- Otherwise treat as a DOI
  if string.match(input, "^10%.") then
    return { type = "doi", id = input }
  end

  return nil
end

-- Fetch metadata from CrossRef for a DOI
function doi_paper.fetch_crossref(doi)
  local url = "https://api.crossref.org/works/" .. doi
  local resp = net.proxyFetch(url, {
    headers = {
      ["Accept"] = "application/json",
      ["User-Agent"] = "SilverBullet-DOI-Paper/1.0 (mailto:silverbullet@example.com)"
    }
  })

  if not resp.ok then
    return nil, "CrossRef API returned status " .. tostring(resp.status)
  end

  local data = resp.body
  local work = data.message

  -- Extract authors
  local authors = {}
  if work.author then
    for _, a in ipairs(work.author) do
      local name = ""
      if a.given then
        name = a.given
      end
      if a.family then
        if #name > 0 then
          name = name .. " "
        end
        name = name .. a.family
      end
      if #name > 0 then
        table.insert(authors, name)
      end
    end
  end

  -- Extract date
  local date_str = ""
  local year = nil
  local date_parts = nil
  if work.published then
    date_parts = work.published["date-parts"]
  elseif work["published-print"] then
    date_parts = work["published-print"]["date-parts"]
  elseif work["published-online"] then
    date_parts = work["published-online"]["date-parts"]
  elseif work.created then
    date_parts = work.created["date-parts"]
  end

  if date_parts and date_parts[1] then
    local parts = date_parts[1]
    year = parts[1]
    if parts[1] then
      date_str = tostring(parts[1])
      if parts[2] then
        date_str = date_str .. "-" .. string.format("%02d", parts[2])
        if parts[3] then
          date_str = date_str .. "-" .. string.format("%02d", parts[3])
        end
      end
    end
  end

  -- Extract venue (container-title is journal/conference name)
  local venue = ""
  if work["container-title"] and #work["container-title"] > 0 then
    venue = work["container-title"][1]
  elseif work["event"] and work["event"].name then
    venue = work["event"].name
  end

  -- Extract title
  local title = ""
  if work.title and #work.title > 0 then
    title = work.title[1]
  end

  -- Extract abstract (CrossRef sometimes includes it with JATS XML tags)
  local abstract = ""
  if work.abstract then
    abstract = work.abstract
    -- Strip JATS XML tags
    abstract = string.gsub(abstract, "<[^>]+>", "")
    abstract = string.trim(abstract)
  end

  -- Extract publisher
  local publisher = work.publisher or ""

  return {
    title = title,
    authors = authors,
    doi = doi,
    url = "https://doi.org/" .. doi,
    venue = venue,
    publisher = publisher,
    date = date_str,
    year = year,
    abstract = abstract,
    source = "crossref"
  }
end

-- Fetch metadata from arXiv API
function doi_paper.fetch_arxiv(arxiv_id)
  local url = "http://export.arxiv.org/api/query?id_list=" .. arxiv_id .. "&max_results=1"
  local resp = net.proxyFetch(url)

  if not resp.ok then
    return nil, "arXiv API returned status " .. tostring(resp.status)
  end

  local body = resp.body

  -- Simple XML extraction helpers (Space Lua doesn't have an XML parser)
  local function extract_tag(xml, tag)
    local pattern = "<" .. tag .. "[^>]*>(.-)</" .. tag .. ">"
    return string.match(xml, pattern)
  end

  local function extract_all(xml, tag)
    local results = {}
    for content in string.gmatch(xml, "<" .. tag .. "[^>]*>(.-)</" .. tag .. ">") do
      table.insert(results, content)
    end
    return results
  end

  -- Find the entry
  local entry = extract_tag(body, "entry")
  if not entry then
    return nil, "No entry found in arXiv response"
  end

  -- Title
  local title = extract_tag(entry, "title") or ""
  title = string.gsub(title, "%s+", " ")
  title = string.trim(title)

  -- Authors
  local authors = {}
  local author_blocks = extract_all(entry, "author")
  for _, block in ipairs(author_blocks) do
    local name = extract_tag(block, "name")
    if name then
      table.insert(authors, string.trim(name))
    end
  end

  -- Abstract
  local abstract = extract_tag(entry, "summary") or ""
  abstract = string.gsub(abstract, "%s+", " ")
  abstract = string.trim(abstract)

  -- Published date
  local published = extract_tag(entry, "published") or ""
  local date_str = string.sub(published, 1, 10) -- YYYY-MM-DD
  local year = nil
  if #date_str >= 4 then
    year = tonumber(string.sub(date_str, 1, 4))
  end

  -- Categories (primary)
  local primary_cat = ""
  local cat_match = string.match(entry, '<arxiv:primary_category[^>]*term="([^"]*)"')
  if cat_match then
    primary_cat = cat_match
  end

  -- Journal ref if available
  local journal_ref = extract_tag(entry, "arxiv:journal_ref") or ""
  journal_ref = string.trim(journal_ref)

  -- DOI if available
  local doi = extract_tag(entry, "arxiv:doi") or ""
  doi = string.trim(doi)
  if #doi == 0 then
    doi = "10.48550/arXiv." .. arxiv_id
  end

  return {
    title = title,
    authors = authors,
    doi = doi,
    url = "https://arxiv.org/abs/" .. arxiv_id,
    venue = journal_ref,
    publisher = "arXiv",
    date = date_str,
    year = year,
    abstract = abstract,
    arxiv_id = arxiv_id,
    arxiv_category = primary_cat,
    source = "arxiv"
  }
end

-- Build frontmatter YAML string from metadata
function doi_paper.build_frontmatter(meta, custom_tags)
  local lines = {}
  table.insert(lines, "---")
  table.insert(lines, 'title: "' .. string.gsub(meta.title, '"', '\\"') .. '"')

  -- Authors as YAML list
  table.insert(lines, "authors:")
  for _, author in ipairs(meta.authors) do
    table.insert(lines, '  - "' .. string.gsub(author, '"', '\\"') .. '"')
  end

  table.insert(lines, 'doi: "' .. meta.doi .. '"')
  table.insert(lines, 'url: "' .. meta.url .. '"')

  if meta.venue and #meta.venue > 0 then
    table.insert(lines, 'venue: "' .. string.gsub(meta.venue, '"', '\\"') .. '"')
  end

  if meta.publisher and #meta.publisher > 0 then
    table.insert(lines, 'publisher: "' .. string.gsub(meta.publisher, '"', '\\"') .. '"')
  end

  if meta.date and #meta.date > 0 then
    table.insert(lines, 'date: "' .. meta.date .. '"')
  end

  if meta.year then
    table.insert(lines, "year: " .. tostring(meta.year))
  end

  if meta.arxiv_id then
    table.insert(lines, 'arxivId: "' .. meta.arxiv_id .. '"')
  end

  if meta.arxiv_category and #meta.arxiv_category > 0 then
    table.insert(lines, 'arxivCategory: "' .. meta.arxiv_category .. '"')
  end

  table.insert(lines, "type: paper")

  -- Tags
  local tags = custom_tags or {"paper", "unread"}
  table.insert(lines, "tags:")
  for _, tag in ipairs(tags) do
    table.insert(lines, "  - " .. tag)
  end

  table.insert(lines, "---")

  return table.concat(lines, "\n")
end

-- Build page body
function doi_paper.build_page(meta, custom_tags)
  local fm = doi_paper.build_frontmatter(meta, custom_tags)
  local body_lines = {}

  if meta.abstract and #meta.abstract > 0 then
    table.insert(body_lines, "## Abstract")
    table.insert(body_lines, "")
    table.insert(body_lines, meta.abstract)
    table.insert(body_lines, "")
  end

  table.insert(body_lines, "## Notes")
  table.insert(body_lines, "")

  return fm .. "\n\n" .. table.concat(body_lines, "\n")
end

-- Read config values
function doi_paper.get_config()
  local cfg = config.get("doiPaper") or {}
  return {
    prefix = cfg.prefix or "Papers",
    tags = cfg.tags or {"paper", "unread"}
  }
end

-- Main command
command.define {
  name = "Paper: Import from DOI",
  run = function()
    local input = editor.prompt("Enter DOI or arXiv ID")
    if not input or #string.trim(input) == 0 then
      return
    end

    local parsed = doi_paper.parse_identifier(input)
    if not parsed then
      editor.flashNotification("Could not parse identifier: " .. input, "error")
      return
    end

    editor.flashNotification("Fetching metadata...")

    local meta, err
    if parsed.type == "arxiv" then
      meta, err = doi_paper.fetch_arxiv(parsed.id)
    else
      meta, err = doi_paper.fetch_crossref(parsed.id)
    end

    if not meta then
      editor.flashNotification("Failed to fetch metadata: " .. (err or "unknown error"), "error")
      return
    end

    if not meta.title or #meta.title == 0 then
      editor.flashNotification("No title found in metadata", "error")
      return
    end

    local cfg = doi_paper.get_config()
    local page_name_base = doi_paper.sanitize_page_name(meta.title)
    local page_name = cfg.prefix .. "/" .. page_name_base

    -- Let user confirm/edit the page name
    page_name = editor.prompt("Page name", page_name)
    if not page_name then
      return
    end

    -- Check if page exists
    if space.pageExists(page_name) then
      editor.flashNotification("Page already exists: " .. page_name, "error")
      return
    end

    -- Build and write the page
    local content = doi_paper.build_page(meta, cfg.tags)
    space.writePage(page_name, content)
    editor.navigate({kind = "page", page = page_name})
    editor.flashNotification("Paper note created!")
  end
}
```
