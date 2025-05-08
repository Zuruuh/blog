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

So, first of all, I initally thought C# could be a great choice because I wanted
my app to feel native. After checking quickly the getting started and all the
crap you need to install to get a hello world, I immediately changed my mind and
went back to my web slop 😬. C# would definitely be the "correct" choice here
but I am confident I can get away with one of the other two options with a
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
