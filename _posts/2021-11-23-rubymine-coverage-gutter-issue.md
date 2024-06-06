---
layout: post
title:  "RubyMine Coverage Gutter Issue (Symlinks!)"
date:   2021-11-23
categories: rubymine
---

I encountered incorrect code coverage highlighting in RubyMine when project was opened via symlink. 

This happens because  The project files were stored in /actual/project/path, but a symlink to this location was created at /symlink/path. Opening the RubyMine project via the symlink caused a mismatch between the real file paths and the paths stored in SimpleCov's coverage data.

Specifically, SimpleCov expanded the paths to their full real locations (e.g. /actual/project/path/file.rb) while RubyMine saw the symlinked path (e.g. /symlink/path/file.rb). This caused RubyMine to be unable to match the files to their coverage data, resulting in missing or incorrect highlighting.

The root cause was accessing the project via symlink rather than the real file path. **To resolve, reopened the project directly via /actual/project/path instead of the symlink.**

