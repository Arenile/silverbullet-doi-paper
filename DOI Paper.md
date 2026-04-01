---
name: Library/doi-paper/DOI Paper
tags: meta/library
---

# DOI Paper Import

A Space Lua library for creating research paper notes from a DOI. Supports papers registered with CrossRef (ACM Digital Library, IEEE Xplore, arXiv, and most other publishers).

## Usage

Run the **Paper: Import from DOI** command from the command palette. Enter a DOI (e.g. `10.1145/3582016.3582063`) or an arXiv ID (e.g. `2301.13808`), and a new page will be created under `Papers/` with frontmatter populated from the paper's metadata.

## Supported Identifiers

| Input Format | Example | Resolution Method |
|---|---|---|
| DOI (bare) | `10.1145/3582016.3582063` | CrossRef REST API |
| DOI (URL) | `https://doi.org/10.1145/...` | CrossRef REST API |
| arXiv ID | `2301.13808` | CrossRef REST API |
| arXiv DOI | `10.48550/arXiv.2301.13808` | CrossRef REST API |

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
    prefix = "Papers",       -- page path prefix (default: "Papers")
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
  local clean = title
  clean = string.gsub(clean, "[/\\#%[%]|{}]", "")
  clean = string.gsub(clean, "%s+", " ")
  clean = string.trim(clean)
  if #clean > 120 then
    clean = string.sub(clean, 1, 120)
  end
  return clean
end

-- Escape a string for safe inclusion inside a YAML double-quoted value.
-- Guards against injection of arbitrary frontmatter keys via crafted
-- paper titles, author names, venue names, etc.
local function yaml_escape(s)
  if not s then return "" end
  s = string.gsub(s, "\\", "\\\\")
  s = string.gsub(s, '"', '\\"')
  s = string.gsub(s, "\n", "\\n")
  s = string.gsub(s, "\r", "\\r")
  s = string.gsub(s, "\t", "\\t")
  return s
end

-- Sanitize a block of text used in the page body (outside frontmatter).
-- Strips lines that could be interpreted as YAML frontmatter delimiters.
local function sanitize_body_text(s)
  if not s then return "" end
  s = string.gsub(s, "\n%-%-%-\n", "\n")
  s = string.gsub(s, "^%-%-%-\n", "")
  s = string.gsub(s, "\n%-%-%-$", "")
  return s
end

-- Detect identifier type from user input
-- Everything is resolved via CrossRef; bare arXiv IDs get a DOI constructed
function doi_paper.parse_identifier(input)
  input = string.trim(input)

  -- Strip URL prefixes
  input = string.gsub(input, "^https?://doi%.org/", "")
  input = string.gsub(input, "^https?://dx%.doi%.org/", "")
  input = string.gsub(input, "^https?://arxiv%.org/abs/", "")

  -- arXiv DOI already (10.48550/arXiv.XXXX.XXXXX)
  if string.match(input, "^10%.48550/arXiv%.") then
    return { type = "doi", id = input }
  end

  -- Bare arXiv ID (YYMM.NNNNN with optional version suffix)
  if string.match(input, "^%d%d%d%d%.%d%d%d%d%d?v?%d*$") then
    local base = string.gsub(input, "v%d+$", "")
    return { type = "doi", id = "10.48550/arXiv." .. base }
  end

  -- Old-style arXiv ID (e.g. hep-th/9901001)
  if string.match(input, "^[a-z%-]+/%d+v?%d*$") then
    local base = string.gsub(input, "v%d+$", "")
    return { type = "doi", id = "10.48550/arXiv." .. base }
  end

  -- Regular DOI
  if string.match(input, "^10%.") then
    return { type = "doi", id = input }
  end

  return nil
end

-- Helper: safely get a value from a table using a key that may contain hyphens
-- After the JSON round-trip this should work with normal bracket access,
-- but we guard against nil at every step to avoid "index a nil value" errors.
local function safe_get(tbl, key)
  if tbl == nil then return nil end
  return tbl[key]
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

  -- Round-trip: the auto-parsed JS object doesn't handle hyphenated keys
  -- (like "date-parts", "container-title") well in Space Lua.
  -- Stringify back to JSON, then re-parse with YAML.parse to get a proper Lua table.
  local body = resp.body
  local body_str
  if type(body) == "string" then
    body_str = body
  else
    body_str = js.JSON.stringify(body)
  end
  local data = YAML.parse(body_str)

  if not data or not data.message then
    return nil, "Unexpected response structure from CrossRef"
  end

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

  -- Extract date: try several fields in order of preference
  local date_str = ""
  local year = nil
  local date_parts = nil

  local published = safe_get(work, "published")
  local published_print = safe_get(work, "published-print")
  local published_online = safe_get(work, "published-online")
  local created = safe_get(work, "created")

  if published then
    date_parts = safe_get(published, "date-parts")
  elseif published_print then
    date_parts = safe_get(published_print, "date-parts")
  elseif published_online then
    date_parts = safe_get(published_online, "date-parts")
  elseif created then
    date_parts = safe_get(created, "date-parts")
  end

  if date_parts and date_parts[1] then
    local parts = date_parts[1]
    year = parts[1]
    if parts[1] then
      date_str = tostring(parts[1])
      if parts[2] then
        local m = tonumber(parts[2])
        if m and m < 10 then
          date_str = date_str .. "-0" .. tostring(m)
        else
          date_str = date_str .. "-" .. tostring(parts[2])
        end
        if parts[3] then
          local d = tonumber(parts[3])
          if d and d < 10 then
            date_str = date_str .. "-0" .. tostring(d)
          else
            date_str = date_str .. "-" .. tostring(parts[3])
          end
        end
      end
    end
  end

  -- Extract venue (container-title is journal/conference name)
  local venue = ""
  local container_title = safe_get(work, "container-title")
  if container_title and #container_title > 0 then
    venue = container_title[1]
  else
    local event = safe_get(work, "event")
    if event then
      venue = safe_get(event, "name") or ""
    end
  end

  -- Extract title
  local title = ""
  if work.title and #work.title > 0 then
    title = work.title[1]
  end

  -- Extract abstract (CrossRef sometimes includes JATS XML tags)
  local abstract = ""
  if work.abstract then
    abstract = work.abstract
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

-- Build frontmatter YAML string from metadata
-- All string values from the API are run through yaml_escape to prevent
-- injection of arbitrary YAML keys via crafted metadata.
function doi_paper.build_frontmatter(meta, custom_tags)
  local lines = {}
  table.insert(lines, "---")
  table.insert(lines, 'title: "' .. yaml_escape(meta.title) .. '"')

  table.insert(lines, "authors:")
  for _, author in ipairs(meta.authors) do
    table.insert(lines, '  - "' .. yaml_escape(author) .. '"')
  end

  table.insert(lines, 'doi: "' .. yaml_escape(meta.doi) .. '"')
  table.insert(lines, 'url: "' .. yaml_escape(meta.url) .. '"')

  if meta.venue and #meta.venue > 0 then
    table.insert(lines, 'venue: "' .. yaml_escape(meta.venue) .. '"')
  end

  if meta.publisher and #meta.publisher > 0 then
    table.insert(lines, 'publisher: "' .. yaml_escape(meta.publisher) .. '"')
  end

  if meta.date and #meta.date > 0 then
    table.insert(lines, 'date: "' .. yaml_escape(meta.date) .. '"')
  end

  if meta.year then
    table.insert(lines, "year: " .. tostring(meta.year))
  end

  table.insert(lines, "type: paper")

  local tags = custom_tags or {"paper", "unread"}
  table.insert(lines, "tags:")
  for _, tag in ipairs(tags) do
    table.insert(lines, "  - " .. tag)
  end

  table.insert(lines, "---")

  return table.concat(lines, "\n")
end

-- Build page body
-- The abstract is sanitized to prevent stray "---" lines from being
-- interpreted as frontmatter delimiters by the markdown parser.
function doi_paper.build_page(meta, custom_tags)
  local fm = doi_paper.build_frontmatter(meta, custom_tags)
  local body_lines = {}

  if meta.abstract and #meta.abstract > 0 then
    table.insert(body_lines, "## Abstract")
    table.insert(body_lines, "")
    table.insert(body_lines, sanitize_body_text(meta.abstract))
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

    local meta, err = doi_paper.fetch_crossref(parsed.id)

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

    page_name = editor.prompt("Page name", page_name)
    if not page_name then
      return
    end

    if space.pageExists(page_name) then
      editor.flashNotification("Page already exists: " .. page_name, "error")
      return
    end

    local content = doi_paper.build_page(meta, cfg.tags)
    space.writePage(page_name, content)
    editor.navigate({kind = "page", page = page_name})
    editor.flashNotification("Paper note created!")
  end
}
```
