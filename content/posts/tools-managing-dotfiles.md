---
title: "Exploring Tools For Managing Your Dotfiles"
date: 2024-12-17T15:14:49+01:00
tags: ["Dotfiles"]
---

It’s been almost 2 years since I installed Arch on my laptop, and I feel like
my dotfiles are becoming more and more disorganized. I still have configuration
files from programs I no longer use, and keeping track of how I customize the
ones I do use is taking far more effort than it should. To address this, I’ve
decided to put my dotfiles under version control, so I can have a clearer view
of how they change over time.

However, with so many options available, picking the right tool for the job
isn’t as straightforward as it seems. This article compares different
solutions, outlining the pros and cons of each, to help you (and me) choose the
one that best fits our needs.

## Two main strategies

A quick search online will provide you with a number of different tools, but
most of them can be categorized into one of two camps:
1. A Git repository in your home directory
2. Symlinks

The first approach is fairly straightforward, as it simply involves placing a
Git repository in your home directory and tracking your dotfiles directly from
there.

The second approach introduces a bit more overhead, as it requires moving all
your dotfiles to a designated directory (which will also house the Git repository) and using symlinks to make them appear where the operating
system expects to find them.

For example, you could move your `.bashrc` to the `~/dotfiles` directory and
then create a symlink that points to it in your home directory using the
following command.

```bash
ln -s ~/dotfiles/.bashrc ~/.bashrc
```

Configuring a new machine from scratch can become quite tedious due to the large number of symlinks involved, but the process can be automated to varying degrees, depending on the tool used.

## Bare Git repository

The most minimal way to turn your entire home directory into a git repository (first approach) is to use a bare Git repository.

[This guide](https://www.atlassian.com/git/tutorials/dotfiles) already provides a pretty clear explanation, so I will just point out that the bare repository (`.cfg` in the tutorial) must not be added to itself. I will also reassure you that the bare repository won't interfere with any other repository you might have inside your HOME directory.

### Pros and cons

This is the simplest approach available, but it comes with the risk of exposing sensitive information by inadvertently pushing files to your potentially public dotfiles repository.

## GNU Stow

One tool for managing symlinks is [GNU
Stow](https://www.gnu.org/software/stow/). To understand how it works, let's
start by defining a few terms:
- A **package** is a collection of files and directories that you want to manage as a unit.
- The **target directory** is where you want all packages to appear to be installed.
- A **stow directory** is the directory containing all your packages.

For each package, Stow creates a symlink in the target directory that points to
the corresponding directory in the stow directory. To make things easier to
understand, let’s look at an example. This is just a quick overview of the main
functionality of the tool to help you determine if it resonates with you. For
more details, feel free to check out the
[docs](https://www.gnu.org/software/stow/manual/stow.html).

### Managing multiple packages

Let's say I want to manage of my `.bashrc` and Neovim config keeping them in
two separate packages for reasons that will become clear in a moment.

```
/home/gb
├── .config
│   └── nvim
│       └── init.lua
└── .bashrc
```

Each package must mimic the tree structure of the target directory, which, in
our case, is the home directory.

```
dotfiles/
├── bash
│   └── .bashrc
└── neovim
    └── .config
        └── nvim
            └── init.lua
```

While in the stow directory (in our case `~/dotfiles`), run `stow neovim` and
`stow bash` to create symlinks the two packages inside the home directory. To
install all packages defined in the stow directory, run `stow *`.

By default the target directory is the parent of the stow directory (the one
        where the `stow` command is run), but this can be overridden with the
`--target` flag.

```
dotfiles
├── .bashrc -> dotfiles/bash/.bashrc
├── .config -> dotfiles/neovim/.config
└── dotfiles
    ├── bash
    │   └── .bashrc
    └── neovim
        └── .config
            └── nvim
                └── init.lua
```

Notice that `~/.config` is a symlink to the `.config` directory inside the
`neovim` package. If we were to add another package that uses the `.config`
directory, `~/.config` would become an actual directory and the symlinks would
be moved into it.

```
/home/gb
├── .bashrc -> dotfiles/bash/.bashrc
├── .config
│   ├── nvim -> ../dotfiles/neovim/.config/nvim
│   └── qtile -> ../dotfiles/qtile/.config/qtile
└── dotfiles
    ├── bash
    │   └── .bashrc
    ├── neovim
    │   └── .config
    │       └── nvim
    │           └── init.lua
    └── qtile
        └── .config
            └── qtile
                └── config.py
```

This happens because Stow minimizes the number of symlinks necessary to mirror
the contents of all packages into the target directory.

### Adding new files

To keep track of new config files, you need to manually move them into your
package of choice.

For example, if I started using Zsh alongside Bash and wanted to keep track of
my `.zshrc` in a new package, I would run the following commands.

```bash
cd ~/dotfiles
mkdir zshell
cp ~/.zshrc zshell
stow zshell --adopt
```

Not using the `--adopt` flag would result in the following message.
```
WARNING! stowing zshell would cause conflicts:
  * cannot stow dotfiles/zshell/.zshrc over existing target .zshrc since neither a link nor a directory and --adopt not specified
```

With the `--adopt` flag, Stow updates the copy of `.zshrc` in the `zshell`
package with the target directory.

This is particularly useful when you are installing your dotfiles on a new
system and your stow directory is under version control because you can run
`git diff` to how the copy in the target directory differs from the one in your
dotfiles repository and then decide how to deal with the changes.

To avoid the conflict all together, you could use `mv` instead of `cp` to move
the `.zshrc` out of the home directory and into the `zshell` package, but I
needed an excuse to tell you about the `--adopt` flag.

### Pros and cons

One of the main benefits of GNU Stow is the ability to independently manage
dotfiles for different programs by keeping them in separate packages. This
allows you to choose which packages you want to clone on each machine to avoid
cluttering up your workspace with useless dotfiles.

While it might sound a bit confusing at first, it becomes pretty intuitive once
you start using it. However, the rule that each package must mimic the original
directory structure will result in a lot of empty directories, which you may
find annoying. For an example of how a dotfiles repositories managed with GNU
Stow looks link, you can take a look at [this
one](https://github.com/xero/dotfiles) on GitHub. In addition, this means that
anybody who might want to use your dotfiles would also have to start using this
tool.

Another disadvantage is the difficulty in migrating away from it. Because the
`stow` command only creates symlinks, you will have to manually move all
dotfiles to their original location.

## YADM

YADM is a Git wrapper designed to keep your dotfiles under version control
without having to create any symlink. Under the hood, it still uses a bare Git
repository, which can be found in `~/.local/share/yadm/repo.git`, but it spares
you the trouble of creating it yourself and defining an alias to go along with
it. 

It also has some more advanced features, such as alternate files, templates,
   encryption, and the the ability to define scripts that run automatically.
   However, I haven't explored any of these features myself, so I recommend
   reading the [official documentation](https://yadm.io/docs/getting_started)
   for more detailed information.

### Ignoring files

By default YADM ignores untracked files when displaying the status. Changing
this behavior is not a good idea as it would significantly slow down the
command.

If you add a `.gitignore` file to your home directory (or its subdirectories),
   YADM will ignore these patters exactly as Git would. Add these
    `.gitignore` files to your repository to have them synced across devices.

Personally, I prefer to have multiple `.gitignore` files inside each
subdirectory instead of a single one in the home directory. For example, I use
this approach to ignore the `__pycache__` directory inside the
[Qtile](https://qtile.org/) config directory.

```
/home/gb/.config/qtile
├── .gitignore <-- "__pycache__"
├── __pycache__
│   └── ...
├── config.py
└── modules
    ├── __init__.py
    ├── __pycache__
    │   ├── ...
    │   └── ...
    ├── common.py
    ├── ...
    └── utils.py
```

Another option is to add patterns to `$HOME/.local/share/yadm/repo.git/info/exclude`
only meant for local configuration but this couldn't be synced across devices
because you cannot add the bare repo to itself

### Adding Git submodules

If you want to add a directory that already contains a Git repository inside of
it, you have to add it as a submodule with the following command

```bash
yadm submodule add <repository_url> <path>
```

where `path` is the directory with the Git repository inside of it and
`repository_url` is the URL to a remote server (like GitHub or GitLab) where
the repository is available so that future clones of the dotfiles repository
will be able to find the submodule and fetch its content.

### Pros and cons

The main benefit of this tool is that it’s essentially just a Git wrapper. Its
main drawback is that it’s just a Git wrapper. On a more serious note, if
you're familiar with Git, you’ll automatically know how to use YADM, but there
are definitely more user-friendly options out there.

Another minor nitpick is that if your dotfiles repository includes a README, it
will clutter up your home directory. I’ve found a potential solution [on
GitHub](https://github.com/yadm-dev/yadm/issues/93#issuecomment-582585718), but
I haven’t managed to implement it yet. I might write a short follow-up post
about it once I get it to work.

## Chezmoi

Chezmoi is a tool written in Go specifically designed for (quoting their
        homepage) managing your dotfiles across multiple diverse machines,
        securely. It's very easy to use and extremely well documented, so I
        will you refer you to the official [quick
        start](https://www.chezmoi.io/quick-start/) guide.

This tool creates a copy of the files you want to track to the
`.local/share/chemoiz` directory, which you can reach via the `chezmoi cd`
command. There are several ways to [edit
files](https://www.chezmoi.io/user-guide/frequently-asked-questions/usage/#how-do-i-edit-my-dotfiles-with-chezmoi),
but the two main strategies are either to use the `chezmoi edit` command, or to
edit the original file and then add it back with the `chezmoi add` command.

Where Chezmoi really shine is managing dotifiles [across multiple
machines](https://www.chezmoi.io/user-guide/manage-machine-to-machine-differences/)
running different operating systems. Also, like YADM, it provides
[scripts](https://www.chezmoi.io/user-guide/use-scripts-to-perform-actions/)
that can run when certain commands are executed and
[encryption](https://www.chezmoi.io/user-guide/encryption/) to protect secrets.
Again, I haven't tested any of these more advanced features as they go beyond
my current needs.

### Ignoring files

To tell Chezmoi to [ignore specific files or
directories](https://www.chezmoi.io/reference/special-files/chezmoiignore/),
you can either use `.chezmoiignore` files, which work almost like `.gitignore`
files except for some minor differences around pattern matching.

A `.chezmoiignore` located in a directory will only be applied from that
directory downwards (just like YADM). Alternatively, you can have a global
`.chezmoiignore` by putting it in the Chezmoi directory. Compared to YADM, this
has the advantage of not cluttering your home directory with a `.gitignore`
file.

### Pros and cons

Chezmoi has a lot going for it, starting with great documentation and a very
active community. It's probably the most user-friendly and feature-rich tool on
this list, so if you need the advanced features it offers or just want to go
with the safest option, this is probably it.

If I had to find a downside, it would probably be the renaming of files with
the `dot_` prefix. Similar to GNU Stow, this forces anyone who wants to use
your dotfiles to install it. The advantage here is that, since the dotfiles are
actual files rather than symlinks, you can stop using Chezmoi at any point
without needing to take any further action.

## Conclusion

As spoiled in the section about it, I’ve already started using YADM because it
seemed like the most minimal solution that doesn’t require me to manually
manage a bare Git repository, which felt a bit hacky. I have a natural tendency
to avoid bloat and prefer the most minimal option available, even if it
requires sacrificing some user-friendliness. After all, I use Arch for a
reason.

Chezmoi feels a bit overkill for the task at hand—at least in my current
situation, where I simply need to manage the dotfiles on my laptop. That said,
    I can see myself adopting this tool when my setup becomes more complex,
    hopefully in the not-too-distant future.

