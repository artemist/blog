---
layout: post
title: What is nixos-rebuild anyway?
date: 2024-01-29
---
If you've used NixOS before, you've almost certainly used the `nixos-rebuild` program before.
With one `nixos-rebuild switch` command you can build your updated system configuration,
add it to your bootloader as the default entry, stop all old services, and start any new services.

What you may not know is that `nixos-rebuild` is a bash script and you can do everything (relatively) easily without it.
The [full source code](https://github.com/NixOS/nixpkgs/blob/c074160dcfa338f8424c440ccb0f0a5412de0dbf/pkgs/os-specific/linux/nixos-rebuild/nixos-rebuild.sh) is quite long and includes many special cases, but most of these aren't necessary if you're building manually.

Some of this will make more sense if you know about two options `--build-host` and `--target-host`.
With these you can build in one computer, copy it to the destination, and install there.

Unforutnately, there's one important question  we have to answer first:

# What is a NixOS?
NixOS is a very complicated way of defining options and setting them to a value.
For example, your configuration you could set:
```
environment.systemPackages = [ pkgs.git ];
```
The value is matched by an "option" of the same name which describes the default value and the type [^type].
This is how NixOS describes `environment.systemPackages`:
```
options.environment.systemPackages = mkOption {
  type = types.listOf types.package;
  default = [];
  example = literalExpression "[ pkgs.firefox pkgs.thunderbird ]";
  description = lib.mdDoc ''
    <removed for brevity>
  '';
};
```

The idea that makes this useful is that you can set values based off of each other.
For example, the gamescope service sets;
```
environment.systemPackages = mkIf (!cfg.capSysNice) [ gamescope ];
```
meaning that `gamescope` is only added to `systemPackages` if the option `programs.gamescope.capSysNice` is disabled.

So we have a big set of options with values, but that doesn't make it an operating system.
The piece that ties this all together is the "toplevel derivation", at `system.build.toplevel`.
The toplevel derivation is a package that takes all of your settings and stuffs them into
one directory. For example, when you set a kernel, the toplevel derivation sees that and puts it in `kernel`.
When a module wants to put a configuration file in `/etc` it creates a file in `etc`.
And all the packages in `environment.systemPackages` get linked together and put into `sw`.
If you want to see yours, then go to `/run/current-system` on a NixOS machine.
This will always have the version you're currently running [^booted-system].


# Step 0: Build nix
The script tries not to make too many assumptions about the build host. It must have a nix store,
but that doesn't necessariy mean it has a new enough nix to build your configuration, or that your configuration
is defined using only settings that your build nix can understand. Therefore, it downloads a newer nix if possible,
or builds one using your configuration. I'm not _entirely_ sure why this happens, but it does.

# Step 1: Building



## Wait what if you're building remotely?
Building a NixOS configuration is secretly two steps: evaluation and realisation.
Evaluation turns your nix code into a derivation: a file with build instructions (environment variables, a build script, and arguments), a list of other derivations its needs before it can build,
and the outputs it will create when you run it ("realise" in nix terminology).
Evaluation is always done on the local machine (where you run `nixos-rebuild`), because the build host might have different channels (for non-flake builds) or no access to the source of some inputs (for flake builds)[^flake-eval].

Realisation takes a derivation and its tree of dependencies then creates the output paths by running each
build script in its own sandbox (or downloading the result from a trusted substituter, often [cache.nixos.org](https://cache.nixos.org/)). Most of the hard work happens here.

In order to get the derivations from the local machine to the build host,
`nixos-rebuild` uses `nix copy --derivation --to`,
which works just like `nix-copy-closure` but copies derivations instead of the entire closure.
Thn the hard work can happen on the destination without needing to copy all the nix source code and dependencies.


# Step 2: Add a profile
# Step 3: Activate

[^booted-system]: ... for a certain definition of "running". Software is loaded from here but the kernel and modules will be in the version in `/run/booted-system` because Linux can't load modules from other kernel versions. This is only setup at boot and won't be changed by a `nixos-rebuild switch`.
[^flake-eval]: It is actually possible to evaluate flakes on the remote machine, but this isn't supported. The `nix flake archive` command, which copies a flake and all of its inputs to the nix store, can copy to another machine with the `--to` argument. Building this way works but I haven't bothered writing a patch.
[^type]: In NixOS `type` defines not just "can I set this to a string or only a list" but also what happens when multiple conflicting options are set. If you set `environment.systemPackages = [ pkgs.git ];` in one file and `environment.systemPackages = [ pkgs.mercurial ];` then the result will be `[ pkgs.git pkgs.mercurial ]` because `listOf` says to merge them.
