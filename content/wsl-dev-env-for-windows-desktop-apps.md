+++
title = "Setting up a linux dev-environment for Windows desktop apps"
date = 2025-05-08
[taxonomies]
tags = ["wsl", "windows", "rust", "nix"]
+++

Hello World 👋

Recently I wanted to get started building Windows apps for fun, and let me tell
you I did NOT enjoy this.

## Desktop framework - Pick your poison

There are multiple existing solutions for building such applications, nowadays
you can use pretty much any languages. The ones that really caught my attention
were mostly:

- [Tauri](https://tauri.app/)
- [Dioxus](https://dioxuslabs.com/)
- [C#](https://dotnet.microsoft.com/en-us/apps/desktop)

Other solutions are probably great I don't care really, I am experienced with
web technologies so if I can avoid learning a new language strictly for this
then it's for the better.

So, first of all, I initially thought C# could be a great choice because I
wanted my app to feel native. After checking quickly the getting started and all
the crap you need to install to get a hello world, I immediately changed my mind
and went back to my web slop 😬. C# would definitely be the "correct" choice
here but I am confident I can get away with one of the other two options with a
native-ish feel while still having good performances and keeping a relatively
small bundle size (insert electron-bloat related pun)

So, in the end, I ended up going with Dioxus mostly because I wanted to
experiment it but Tauri would have worked just as good really. Also I think
Dioxus is pretty cool and seeing the updates and breakthrough they made related
to wasm patching for hot reloading on twitter really amazed me so I guess why
not check it out right ?

## Setting up the project

I use [WSL](https://learn.microsoft.com/en-us/windows/wsl/install) as my daily
driver, using [NixOS WSL](https://github.com/nix-community/NixOS-WSL), and
honestly it's the best you could hope from both worlds. It's obviously not
perfects and there are some drawbacks (and we'll talk about them shortly) but
for my day-to-day work in web development it does the job pretty good.

So, as I'm setting up my initial Dioxus project, I grab a quick nix flake from
an existing Tauri project I had and try running it, and yeah, it _works_. Really
? Well, if you already ran some GUI apps through WSL, you might know it does
indeed _work_ (at least if you're on Windows 11), but the experience is complete
garbage, the apps doesn't scale correctly to your monitor size, WebKit GTK is
still as antiquated as ever, etc... In short, it's not as great as you might
think. And, on top of all of this, I suddenly realize

> Wait, I want my app to run on windows because I need some features and
> external programs only available on this platform, but right now it's running
> on linux !?

Yeah, as simple as that, you can't develop on Linux if you target Windows (duh).
Sound obvious right ? But actually no it doesn't.

## The development environment

Actually, I can simply write the code on Linux and run it on Windows right ? It
shouldn't be **that** complicated to setup ? Oh boy if only I knew.

On my first try, I simply tried accessing my project (which is stored in my
Linux drive) from my Windows environment. All WSL instances are automatically
forwarded to a network-mounted drive located at `\\wsl.localhost\<name>`. I
tried to `cd` into it from Windows on [`nushell`](https://www.nushell.sh/), and
yeah, apart from the constant errors telling me that my path is invalid, I
couldn't get any cargo command to run correctly. I'm no expert in this field so
really I just assumed this wasn't doable and went another way.

On my second try, I did the opposite. I stored my project on windows, then
accessed it through the mounted drives located at `/mnt/<drive>` in WSL. And
actually this time it _worked_, meaning I could get the project to open in
neovim from linux, while still being able to build it from Windows! Problem
solved then right ? **WRONG**

## Introducing NTFS

Re-phrasing wikipedia, <u>[NTFS](https://en.wikipedia.org/wiki/NTFS) is a
proprietary journaling file system developed by Microsoft in the 1990s</u>. This
is basically (simplifying) the "format" that the OS uses to stores data on your
hard drives. If you are using windows, chances are that all your drives are
using NTFS.

BUT it is NOT the case for Linux distributions installed through WSL! These ones
use the [Ext4](https://en.wikipedia.org/wiki/Ext4) format (at least on wsl2,
can't say for sure for wsl1). And this is a really important thing to note.
While Linux does support reading and writing to/from NTFS file systems, it
introduces a significant overhead for all FS operations. And you know what
software does a lot of IO in development ? Our best friends, cargo and git
😼😼😼

We are now at a whopping ~5 seconds for a simple `git status`, and around a few
more for a simple `rust-analyzer` completion after a modification (and probably
a few minutes of build with a cold cache). Keep in mind that this is an empty
project! A dead simple hello world with a single dependency being dioxus. I hate
garbage software and there is no way I could work in such conditions.

## Is it really worth going further ?

Now, I had two options. Either I keep banging my head in the walls like a retard
until I find a solution that works. OR I install VSC*de on my windows machine
and use it instead. The answer should be obvious but it did really cross my
mind. Though I can hardly imagine doing anything useful with a terminal that
takes 5 seconds to open and a sluggish web slop IDE like I used to in the past.

So in the end, obviously, I kept my nix-neovim-zellij-nushell setup I love so
much and tried looking for solutions to speed up my setup.

## Going down the performance rabbit hole

Now I knew the two software that were slowing me down were Cargo (and
rust-analyzer) and Git. As said earlier, the reason why they are slow in this
context is because my project is located in `/mnt/z/`, which uses NTFS, and
since both rely a lot on the filesystem, they get a massive performance penalty
because of the file system overhead. So what could be the solution then ?

### Moving out | Cargo

As stupid as it may sound, you can actually locate your `target` directory
outside of your project, in a completely different drive actually. This is
perfect in this case, we can simply configure cargo to always use a target
directory located elsewhere and we're done. Easy win!

We can do so using the
[`CARGO_TARGET_DIR`](https://doc.rust-lang.org/cargo/reference/build-cache.html)
environment variable, which will be trivial to add using
[`direnv`](https://direnv.net/)'s `.envrc`.

```bash,name=.envrc
use flake
export CARGO_TARGET_DIR="${XDG_CACHE_HOME:-"$HOME/.cache"}/.cache/myproject"
```

> I choose not to use nix flakes heres because `devShell.env.*` is supposed to
> be pure so it cannot interpolate the host's environment variables Also I don't
> want to use `shellHook`

And now if I try to `cargo build` a simple default debug build with cold cache:

```sh
$ cargo clean
... 
$ cargo build
...
Compiling myproject-desktop v0.1.0 (/mnt/z/my-project)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 33.45s
```

Astonishing news! Down to a much more reasonable 33 seconds! This means
rust-analyzer will be much quicker to provide analysis and completions, which
are pretty important in rust projects, even more with the extensive use of
macros like it's the case in Dioxus.

But we aren't out of misery just yet. If I try to change a single line in my
`src/main.rs` (Changing the heading from `Hello, World` to `Goodbye, World`) and
try to `cargo build` again with a hot cache, we get:

```sh
$ cargo build
   Compiling myproject-desktop v0.1.0 (/mnt/z/my-project)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 3.16s
```

A whopping 3 seconds of rebuild with a cold cache for a single string change!
This could be good enough but of course we can dive further into it and make it
a lot quicker.

The reason why cargo (actually rustc) takes so much time to recompile a single
change is because of linking. It's a pretty slow part that needs to happen at
the end of each compilation to produce a final usable output (a binary for
example). By default, rustc uses `rustc-lld` when using the `nightly` toolchain
AND the `x86_64-unknown-linux-gnu` triple according to
[this blog post](https://blog.rust-lang.org/2024/05/17/enabling-rust-lld-on-linux/)
dating from March 2024, but historically, you'd use GNU LD by default. And, as
much as it work, it is still pretty damn slow and is an inevitable part of the
build process that we cannot avoid or cache efficiently.

#### Introducing [`mold`](https://github.com/rui314/mold)

<u>Mold</u> is an alternative linker that claims to be ~10x faster than GNU LD!

![Mold benchmarks](/mold_benchmarks.svg)

<center><small>Mold benchmarks against GNU LD, GNU Gold, LLVM LLD</small></center>

Okay it doesn't actually claim to be 10 times faster, but in practice it looks
like it is, and the benchmarks seems really promising.

We can start using it pretty easily, simply by adding it to our nix flake:

```nix,name=flake.nix
pkgs.mkShell {
    nativeBuildInputs = [
        # ... 
        pkgs.mold
    ];
}
```

And now, introducing the cargo configuration file, located at
`.cargo/config.toml` (inside of the project root, not in your `$HOME` dir). Here
we can add arbitrary rustflags that will be added by cargo to all `rustc` calls.
We can now simply configure the `link-arg=-fuse-ld` rustc flag in our new
configuration file just like so:

```toml,name=.cargo/config.toml
[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "link-arg=-fuse-ld=mold"]
```

> Notice that we enable `mold` only on the `x86_64-unknown-linux-gnu` triple,
> since it is not available on windows (yet?) we'll have to stick to the default
> linker for now (TODO: check if we can use rust-lld here on windows?).

And that's it! We can now try to re-compile the project with a simple change
and... 🥁🥁🥁:

```sh
$ cargo build
   Compiling myproject-desktop v0.1.0 (/mnt/z/my-project)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.61s
```

🥳🎉We now have a pretty decent compile time for tinkering locally. We are now
done with cargo-related stuff.

### Moving out | Git

Now, we'll have to apply the same logic to git: move all it's storage (the
`.git` dir) to another drive, while still having access to all git-related
tooling. After a quick research, I learn about the `--git-dir` and `--work-tree`
git flags, which is exactly what I need. After a bit of tinkering with nix's
`makeShellWrapper`, I realized that you can also configure these options through
the environment variables `GIT_WORK_TREE` and `GIT_DIR`, which are a lot simpler
to use and setup. Just like before, we can add them to our `.envrc`

```bash,name=.envrc
use flake
export CARGO_TARGET_DIR="${XDG_CACHE_HOME:-"$HOME/.cache"}/.cache/myproject"
export GIT_DIR="$HOME/dev/my-project.git"
export GIT_WORK_TREE="$PWD"
```

The `GIT_WORK_TREE` is always the working directory, which in the context of
direnv will always be the directory where the `.envrc` is located, so this is
good 👍. Though for the `GIT_DIR`, I wasn't exactly sure where to put it so I
just put some random crap, but you could do better I guess (I didn't look much
into it). With this out of the way, I can now `mv .git ~/dev/my-project.git` and
try to run a git status, and, drum rolls again 🥁🥁🥁:

```sh
$ hyperfine -N 'git status -uno'
Benchmark 1: git status -uno
  Time (mean ± σ):     318.2 ms ±  16.8 ms    [User: 8.6 ms, System: 5.7 ms]
  Range (min … max):   297.9 ms … 351.3 ms    10 runs
```

Honestly this is better than expected, though we can try squeezing a bit more
perfs out of this. We can use many different git options that help a bit

```toml,name=.git/config
[core]
    # Default configuration options
    repositoryformatversion = 0
    bare = false
    logallrefupdates = true
    ignorecase = true

    # https://git-scm.com/docs/git-config#Documentation/git-config.txt-coretrustctime
    trustctime = false
    # https://git-scm.com/docs/git-config#Documentation/git-config.txt-corecheckStat
    checkStat = minimal
    # https://git-scm.com/docs/git-config#Documentation/git-config.txt-corefileMode
    fileMode = false
```

```sh
$ hyperfine -N 'git status -uno'
Benchmark 1: git status -uno
  Time (mean ± σ):     161.4 ms ±   8.0 ms    [User: 4.4 ms, System: 3.2 ms]
  Range (min … max):   153.4 ms … 178.0 ms    17 runs
```

This is pretty much the best we can hope to get. It's still astonishingly long
for a simple git status, but way better than the initial results!

## Fixing nix's flake development time

And even after all of this, when I thought I was done, another issue arises.
This time, it's my nix flake taking multiple minutes to activate a simple
`devShell`. This is a well known problems of flakes, simply put, whenever you
try to build it from scratch, the entire directory will be copied to the nix
store. Even if the input is irrelevant, and there's no way to disable this
behaviour.

If you already used flakes, you might know that this is a thing, but why would
it be a problem right ? Personally, I've never really had any issues with this,
simply because nix adapts to your git repository. That's the reason why it's not
as slow usually, because it only copies files that are tracked by git.

So, with our new garbage setup, nix does not recognize our project as a git
repository, so it proceeds to copy the entirety of it to the nix store, which is
painfully slow since it will copy all the `target` directory from a filesystem
to another. On top of that, it will take humongous amounts of storage for
virtually nothing useful. Let's fix that:

There is a simple workaround we can use, see, our flake doesn't need to know
about the rest of our project, really it could be a standalone thing to simply
install system dependencies. So, we can simply put all our nix-related files in
a `./nix` subdirectory, and update the `.envrc` to use the flake located in this
directory.

```sh,name=.envrc
use flake ./nix
# ...
```

And we now have a development environment that boots in less than a second!

## Takeways

We end up with a dual development environment, with the power of nix and the
native-ness of windows 🙂 Would I recommend this to anyone ? Probably not,
VSCode would work better and you'd save yourself some headaches, but I found it
pretty funny in the end. I simply have to run the project from windows and I get
this pretty cool Dioxus console to monitor my project.

![Dioxus Console](/dioxus_dev_console.png)
