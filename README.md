
The code and queries etc here are unlikely to be updated as my process
evolves. Later repos will likely have progressively different approaches
and more elaborate tooling, as my habit is to try to improve at least
one part of the process each time around.

---------

Step 1: Configure config.json
=============================

All the relevant metadata now lives in config.json: ideally nothing will
need tweaked after this. We need to be careful here to get the history
of Wikidata IDs for the constituency correct.

Step 1: Scrape the results
==========================

```sh
jq -r .wikipedia config.json | xargs bundle exec ruby scraper.rb | tee wikipedia.csv
```

Step 2: Check for missing party IDs
===================================

```sh
xsv search -v -s party 'Q' wikipedia.csv
```

Two un-linked independents (now fixed) plus one listed as L.A.P.P, which
seems to be the "Let's Have Another Party Party", which I've created as Q98543370

Ste 3: Check for missing election IDs
=====================================

```sh
xsv search -v -s election 'Q' wikipedia.csv | xsv select electionLabel | uniq
```

4 missing by-elections created:
* "1899 Exeter by-election" -> Q98543450
* "1844 Exeter by-election" -> Q98543460
* "1845 Exeter by-election" -> Q98543467
* "1864 Exeter by-election" -> Q98543473


Step 4: Generate possible missing person IDs
============================================

```sh
xsv search -v -s id 'Q' wikipedia.csv | xsv select name | tail +2 |
  sed -e 's/^/"/' -e 's/$/"@en/' | paste -s - |
  xargs -0 wd sparql find-candidates.js |
  jq -r '.[] | [.name, .item.value, .election.label, .constituency.label, .party.label] | @csv' |
  tee candidates.csv
```

Step 5: Combine Those
=====================

```sh
xsv join -n --left 2 wikipedia.csv 1 candidates.csv | xsv select '10,1-8' | sed $'1i\\\nfoundid' | tee combo.csv
```

Step 6: Generate QuickStatements commands
=========================================

```sh
bundle exec ruby generate-qs.rb config.json | tee commands.qs
```

Then sent to QuickStatements as https://editgroups.toolforge.org/b/QSv2T/1598017399334/
