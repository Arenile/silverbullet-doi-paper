---
name: Library/doi-paper/DOI Paper
tags: meta/library
---

# DOI Paper Import

A Space Lua library for creating research paper notes from a DOI. Supports papers from CrossRef-registered publishers (ACM Digital Library, IEEE Xplore, etc.) and arXiv preprints (via DataCite).

## Usage

Run the **Paper: Import from DOI** command from the command palette. Enter a DOI (e.g. `10.1145/3582016.3582063`) or an arXiv ID (e.g. `2301.13808`), and a new page will be created under `Papers/` with frontmatter populated from the paper's metadata.

## Supported Identifiers

| Input Format | Example                       | Resolution Method |
| ------------ | ----------------------------- | ----------------- |
| DOI (bare)   | `10.1145/3582016.3582063`     | CrossRef REST API |
| DOI (URL)    | `https://doi.org/10.1145/...` | CrossRef REST API |
| arXiv ID     | `2301.13808`                  | DataCite REST API |
| arXiv DOI    | `10.48550/arXiv.2301.13808`   | DataCite REST API |

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

config.define("doiPaper", {
  description = "DOI Paper Import settings",
  type = "object",
  properties = {
    prefix = { type = "string" },
    tags = { type = "array", items = { type = "string" } }
  },
  additionalProperties = false
})

-- Escape a string for safe inclusion inside a YAML double-quoted value.
local function yaml_escape(s)
  if not s then return "" end
  s = string.gsub(s, "\\", "\\\\")
  s = string.gsub(s, '"', '\\"')
  s = string.gsub(s, "\n", "\\n")
  s = string.gsub(s, "\r", "\\r")
  s = string.gsub(s, "\t", "\\t")
  return s
end

-- Sanitize body text to prevent stray "---" from being parsed as frontmatter.
local function sanitize_body_text(s)
  if not s then return "" end
  s = string.gsub(s, "\n%-%-%-\n", "\n")
  s = string.gsub(s, "^%-%-%-\n", "")
  s = string.gsub(s, "\n%-%-%-$", "")
  return s
end

-- Zero-pad a number to two digits.
local function pad2(n)
  n = tonumber(n)
  if not n then return "00" end
  if n < 10 then return "0" .. tostring(n) end
  return tostring(n)
end

-- Sanitize a title for use as a page name.
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

-- Detect identifier type from user input.
-- Returns { type = "crossref"|"datacite", id = "..." } or nil.
function doi_paper.parse_identifier(input)
  input = string.trim(input)

  -- Strip URL prefixes
  input = string.gsub(input, "^https?://doi%.org/", "")
  input = string.gsub(input, "^https?://dx%.doi%.org/", "")
  input = string.gsub(input, "^https?://arxiv%.org/abs/", "")

  -- arXiv DOI (10.48550/arXiv.XXXX.XXXXX) -> DataCite
  if string.match(input, "^10%.48550/arXiv%.") then
    return { type = "datacite", id = input }
  end

  -- Bare arXiv ID (YYMM.NNNNN with optional version) -> construct DOI, use DataCite
  if string.match(input, "^%d%d%d%d%.%d%d%d%d%d?v?%d*$") then
    local base = string.gsub(input, "v%d+$", "")
    return { type = "datacite", id = "10.48550/arXiv." .. base }
  end

  -- Old-style arXiv ID (e.g. hep-th/9901001)
  if string.match(input, "^[a-z%-]+/%d+v?%d*$") then
    local base = string.gsub(input, "v%d+$", "")
    return { type = "datacite", id = "10.48550/arXiv." .. base }
  end

  -- Regular DOI -> CrossRef
  if string.match(input, "^10%.") then
    return { type = "crossref", id = input }
  end

  return nil
end

-- Fetch metadata from CrossRef for a DOI (ACM, IEEE, etc.)
-- The auto-parsed JSON body is a JS object; bracket access works for
-- hyphenated keys like "container-title" and "date-parts".
function doi_paper.fetch_crossref(doi)
  local url = "https://api.crossref.org/works/" .. doi
  local resp = net.proxyFetch(url, {
    headers = {
      ["Accept"] = "application/json",
      ["User-Agent"] = "SilverBullet-DOI-Paper/1.0 (mailto:silverbullet@example.com)"
    }
  })

  if not resp or not resp.ok then
    local status = resp and tostring(resp.status) or "no response"
    return nil, "CrossRef API returned status " .. status
  end

  local data = resp.body
  if not data or not data.message then
    return nil, "Unexpected response structure from CrossRef"
  end
  local work = data.message

  -- Extract authors
  local authors = {}
  if work.author then
    for _, a in ipairs(work.author) do
      local name = ""
      if a.given then name = a.given end
      if a.family then
        if #name > 0 then name = name .. " " end
        name = name .. a.family
      end
      if #name > 0 then table.insert(authors, name) end
    end
  end

  -- Extract date from various CrossRef date fields
  local date_str = ""
  local year = nil
  local date_parts = nil

  local function try_date(obj, key)
    if obj and obj[key] then return obj[key]["date-parts"] end
    return nil
  end

  date_parts = try_date(work, "published")
    or try_date(work, "published-print")
    or try_date(work, "published-online")
    or try_date(work, "created")

  if date_parts and date_parts[1] then
    local parts = date_parts[1]
    year = parts[1]
    if parts[1] then
      date_str = tostring(parts[1])
      if parts[2] then
        date_str = date_str .. "-" .. pad2(parts[2])
        if parts[3] then
          date_str = date_str .. "-" .. pad2(parts[3])
        end
      end
    end
  end

  -- Extract venue
  local venue = ""
  local ct = work["container-title"]
  if ct and #ct > 0 then
    venue = ct[1]
  elseif work["event"] and work["event"].name then
    venue = work["event"].name
  end

  -- Extract title
  local title = ""
  if work.title and #work.title > 0 then
    title = work.title[1]
  end

  -- Extract abstract (strip JATS XML tags)
  local abstract = ""
  if work.abstract then
    abstract = string.gsub(work.abstract, "<[^>]+>", "")
    abstract = string.trim(abstract)
  end

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
  }
end

-- Fetch metadata from DataCite for a DOI (arXiv papers).
-- DataCite JSON:API structure: data.attributes.{creators, titles, ...}
function doi_paper.fetch_datacite(doi)
  local url = "https://api.datacite.org/dois/" .. doi
  local resp = net.proxyFetch(url, {
    headers = {
      ["Accept"] = "application/json"
    }
  })

  if not resp or not resp.ok then
    local status = resp and tostring(resp.status) or "no response"
    return nil, "DataCite API returned status " .. status
  end

  local body = resp.body
  if not body or not body.data or not body.data.attributes then
    return nil, "Unexpected response structure from DataCite"
  end
  local attrs = body.data.attributes

  -- Extract title
  local title = ""
  if attrs.titles and #attrs.titles > 0 then
    title = attrs.titles[1].title or ""
  end

  -- Extract authors (DataCite calls them "creators")
  local authors = {}
  if attrs.creators then
    for _, c in ipairs(attrs.creators) do
      local name = ""
      if c.givenName then name = c.givenName end
      if c.familyName then
        if #name > 0 then name = name .. " " end
        name = name .. c.familyName
      end
      -- Fall back to the combined "name" field
      if #name == 0 and c.name then
        name = c.name
      end
      if #name > 0 then table.insert(authors, name) end
    end
  end

  -- Extract year
  local year = nil
  if attrs.publicationYear then
    year = tonumber(attrs.publicationYear)
  end

  -- Extract date from "dates" array
  local date_str = ""
  if attrs.dates then
    for _, d in ipairs(attrs.dates) do
      if d.date and #d.date >= 10 then
        date_str = string.sub(d.date, 1, 10)
        break
      elseif d.date and #d.date >= 4 then
        date_str = d.date
        break
      end
    end
  end
  if #date_str == 0 and year then
    date_str = tostring(year)
  end

  -- Extract abstract from descriptions
  local abstract = ""
  if attrs.descriptions then
    for _, desc in ipairs(attrs.descriptions) do
      if desc.description and #desc.description > 0 then
        abstract = desc.description
        abstract = string.gsub(abstract, "<[^>]+>", "")
        abstract = string.trim(abstract)
        break
      end
    end
  end

  -- Publisher: can be a string or an object with a "name" field
  local publisher = ""
  if attrs.publisher then
    if type(attrs.publisher) == "string" then
      publisher = attrs.publisher
    elseif attrs.publisher.name then
      publisher = attrs.publisher.name
    end
  end

  -- Venue: DataCite container may have a title
  local venue = ""
  if attrs.container and attrs.container.title then
    venue = attrs.container.title
  end

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
  }
end

-- Build frontmatter YAML string from metadata.
-- All string values are escaped to prevent YAML injection.
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

-- Build page body.
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

-- Read config values.
function doi_paper.get_config()
  local cfg = config.get("") or {}
  return {
    prefix = cfg.prefix or "Papers",
    tags = cfg.tags or {"paper", "unread"}
  }
end

-- Main command.
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
    if parsed.type == "datacite" then
      meta, err = doi_paper.fetch_datacite(parsed.id)
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
