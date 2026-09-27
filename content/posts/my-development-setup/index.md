+++
title = 'My development setup. A pivot toward terminal and dotfiles'
date = 2026-09-27T00:00:00+01:00
draft = false
tags = ['dotfiles', 'helix', 'zellij', 'alacritty', 'rust', 'terminal', 'workflow']
+++

# My development setup. A pivot toward terminal and dotfiles

I have never been a terminal guy.

I used to use it occasionally but always pivoted to the full-featured environments stuffed with all conceivable tools, themes and God knows what else.

My main IDEs were ones from the Jetbrains family, such as Pycharm, Rustrover, CLion and others alongside VScode and, at the dawn of my career, Eclipse and Visual Studio.

Not to say I did not want to learn Vim or even Emacs (I always secretly envied the people working in vim) but it was always painful.

Either I did not have enough endurance to overcome the adaptation period, gain my muscle memory or at some point I did not see the real benefits of it.

I persuaded myself that if you did not start with Vim/Emacs early on it was too late. It still sounds plausible, though.

But recently, something has changed. I can't describe it rationally but emotionally, my work with some code flows became heavy and cumbersome.

My soul craved something new. Also, seeing PyCharm eating 10GB of my RAM under some load, was definitely not very inspiring.

You can call it irrational (midlife crisis?) but some day I stumbled upon Neovim and then [Helix](https://helix-editor.com/). It dawned on me to try them out.

Also, in the back of my mind, there was a lingering thought that I was missing some details regarding setup, installation or whatever the environment was handling behind the scenes.

My hypothesis was that the way of thinking depends on the tools to some extent and if we change them it influences the way we think and approach the problems.

So, spoiler alert I can second that.

So, since I have been working in the field for a long time with different languages I understand that I could not replace the whole IDE with one single alternative.

It would have to be a composite setup, including several elements.

Fortunately, I had a couple of free evenings.

Also, some of my choices might seem unconventional but just bear in mind, that I love and use Rust as my main language for my projects so if something is written in Rust I will naturally pick it over the alternatives.

It all started with the editor. I considered Vim, but honestly, when I took a look at [Helix](https://helix-editor.com/), it just felt more natural and simpler and, of course, it's written in Rust.

Next came a terminal multiplexer. I picked [Zellij](https://zellij.dev/) as it has good reviews in general and again, written in Rust. From there, the choice of a terminal emulator was obvious: [Alacritty](https://alacritty.org/) (which, as you might guess, is also written in Rust).

The core setup seemed easy enough but figuring out how to consolidate it across the several machines led me to the concept of [dotfiles](https://dotfiles.github.io/). The idea is relatively simple: you drop all the configuration files for your environment in a Git repository, and then use symlinks to place them exactly where they need to be. Elegant. Wrap it all in one `install.sh` file and you get a universal method to deploy the setup to any Mac or Linux machine.

My setup with a closer look:

**The light theme.** This major transition also inspired me to switch to a light theme.

**The primary languages.** Rust, Python, Zig, C++ and C with a focus on Rust and Python.

**The core workspace.** The main tabs are File Manager, Editor, several consoles, an LLM window and a Git flow. The file manager can come in handy but I primarily rely on awesome Helix built-in navigation.

**The layouts.** There are three layouts: `rsdev`, `pydev` and `dev` where they mostly differ by the number of terminals in one tab (I am still experimenting with this)

**The installation script.** One `install.sh` takes care of everything (I hope)

**[Bash](https://en.wikipedia.org/wiki/Bash_(Unix_shell)) is my default shell.**

**The other editors on demand.** For certain tasks, like Python notebooks, VS Code or Pycharm is superior. Therefore, I need a quick way to jump into them.

**Customizable LLM providers.** I work with different providers at my day job and on my personal projects, so having this easily customizable is a necessity.

## What I like

**The fuzzy test runners (`rt` for Rust, `ptk` for Python).** This is the first thing you notice when you separate running from editing. At first, it feels awkward, but it quickly becomes natural.

**Lazygit.** It is just an amazing tool, easy to use, powerful and very visual.

**Helix built-in navigation.** The flat file manager, grep search are just so fast and intuitive that I quickly switched to using them exclusively.

**Attachable sessions.** Every environment (`rsdev`, `pydev`, or `dev`) runs on a single session, and all reconnects seamlessly attach to that same session.

**Zoxide.** It just works and completely replaces `cd` for the routine operations.

## What I am still learning / struggling with

**Quick navigation in file.** The editor provides a marvelous way to quickly navigate open files, but it is completely different from what I am used to. I hope it is just a matter of time.

**Refactoring and code analysis.** The environment is outstanding for Rust but not so bright for Python. When it comes to a deep analysis of usages, inheritance, sometimes I switch to the alternatives.

**Python Notebooks.** PyCharm or VsCode is just better now and super easy to use.

## Overall

Overall, I can say that after a month of use, I am content with my current setup. I am still using PyCharm for some Python tasks, but the lion's share of my work happens in my `rsdev` or `pydev` sessions.

What I want to highlight:

**It changed how I navigate the code base.** I started thinking of the codebase as a flat structure rather than a hierarchy of folders.

**All hidden operations are visible now.** I had to set up all operations and preparations by myself.

**It is extremely lightweight.** Just fast and direct feedback from the tools.

Of course, there is a fly in the ointment. Full IDEs just provide richer functionality for certain tasks. For instance markdown.

It is not an easy task to render and preview it in the console. But I realized it is perfectly normal to have alternative tools for specific jobs.

**And here are my [dotfiles](https://github.com/besok/dotfiles)**
