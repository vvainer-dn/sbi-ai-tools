# Slug conventions — sbi-ai-tools wiki

A **slug** is the page's filename without `.md` — the identifier every
`[[wikilink]]` points at. If slugs are unpredictable, links break and the wiki
graph fragments. These rules keep every link resolvable.

## Rules

1. **Every `[[X]]` must match a real file slug** in `wiki/` (bare slug resolves
   vault-wide; `[[folder/slug]]` also works). Validate before pushing.
2. **Canonical slug format per type:**

   | type | slug format | example |
   |---|---|---|
   | architecture | `<facet-kebab>` | `sbi-control-data-plane` |
   | operations | `<topic>-<playbook-kind>` | `sbi-debugging-and-fmea` |
   | pattern | `<descriptive-kebab>` | `pulsar-compacted-topic-stale-config` |
   | incident | `<topic>-<YYYY-MM-DD>` | `controller-failover-loop-2026-09-01` |
   | resolved_ticket | `ar-<NNNNN>` | `ar-55722` |
   | service | `<service-name-kebab>` | `sbi-controller`, `tcs-subscriptions` |

3. **Never wikilink external repos.** `[[drivenets/dap]]` is wrong; write
   `[drivenets/dap](https://github.com/drivenets/dap)`. Same for PRs, Jira, and
   any URL — markdown links only. Wikilinks are reserved for pages in this vault.
4. **Formatting:** kebab-case, lowercase, no dots, max 100 characters.
5. **Stability:** never rename a slug once other pages link to it; if a rename is
   unavoidable, update every inbound `[[wikilink]]` in the same commit.

## Quick validation

```bash
# list wikilinks that do not resolve to a file in wiki/
grep -rhoE '\[\[[^]|#]+' wiki --include='*.md' | sed 's/\[\[//' | sort -u |
while read -r slug; do
  find wiki -name "$(basename "$slug").md" | grep -q . || echo "BROKEN: [[$slug]]"
done
```
