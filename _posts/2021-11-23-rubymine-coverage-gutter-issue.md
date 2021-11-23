---
layout: post
title:  "RubyMine Coverage Gutter Issue (Symlinks!)"
date:   2021-11-23
categories: rubymine
---

Twice now I've opened up RubyMine run my rails/minitest/simple-cov coverage and while the statistics in the file selector side bar are correct, all gutter highlighting (the highlighting next to the line numbers) was red.

On my laptop I keep an encrypted volume where my source is kept, for safety. Let's call it `/mnt/encrypted-src`.
For convenience I keep a symlink to it in `~/work`. Now most everything works fine if I open a project via the ~/work symlink, but SimpleCov's `coverage/.resultset.json` expands the file refences to their full disk path (removing the symlink) e.g. /mnt/encrypted-src/example.rb. But the RubyMine gutter code must be looking at the symlinked path /home/andy/work/example.rb. So the two don't match, and I get no coverage statistics.


<b>[SOLUTION] **Symlinks! You should not open the project through RubyMine via a symlink. Fix the issue by re-opening the project at it's hard location on disk.**</b>



