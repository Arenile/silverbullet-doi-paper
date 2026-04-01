# SilverBullet Paper Note from DOI

## DISCLAIMER

> This was written almost entirely using Claude Opus 4.6 and at time of writing
> I have relatively little knowledge of Space Lua (SilverBullet's Lua implementation).
> I have attempted to do the bare-minimum to ensure that paper metadata pulled into
> YAML frontmatter is sanitized to at least make sure escape sequences don't work, but
> obviously I can't make any guarantees that this is 100% safe to use. You can substantially
> reduce the attack surface here by only using DOIs directly from trusted sources, though.

## Installation

In your SilverBullet instance just run the `Library: Install` command and when asked
for a URI you want to point it to the `DOI Paper.md` file in this repo.

```
https://github.com/Arenile/silverbullet-doi-paper/blob/main/DOI%20Paper.md
```

## Description and Usage

This is a very simple Library for [SilverBullet](https://silverbullet.md/) v2 that provides the
`Paper: Import from DOI` command. This command lets you put in a DOI string registered with
either DataCite (backs arXiv) or CrossRef (backs IEEE Explore and ACM Digital Library) and it
will pull metadata about that paper from the appropriate API and create a new note
in `Papers/{PAPER TITLE}` (location can be configured) with frontmatter filled in
based on the metadata.

### Example valid input strings

```
10.1145/2000064.2000108
https://doi.org/10.1145/2000064.2000108
```
