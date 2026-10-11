# Colibri — instructions for agents and contributors

- **Language.** Everything in this repository is written in English: documents, code,
  identifiers, comments, commit messages, log lines. The canonical language of a
  diagnostic is the language of the person on call.
- **Documents.** `docs/README.md` indexes them. A document is either *design*
  (authoritative, kept current) or a *journal* (closed, never rewritten). What was not
  measured is written down as not measured: a number is cited with its source or marked
  as a *default* or *target*. A decision names the alternatives it rejected.
- **Ledgers.** An exclusion in a compatibility ledger carries a reason, never a name alone.
- **Public text.** Nothing in this repository links to or names a private repository or
  an internal document. If a claim needs such a source, state the claim without the
  citation or mark it *not measured here*.
- **Dependencies.** Every dependency's licence must be in `deny.toml`'s allow list; a new
  dependency names its licence and reason in the pull request.
- **Commits.** Commit messages and pull request descriptions carry no attribution
  trailers or generated-by lines (no `Co-Authored-By`, no "Generated with"). The author
  field is the person who committed.
