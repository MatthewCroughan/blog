+++
title = "Godot's Export to Android: Easy with Nix"
date = 2022-01-06
cover = "./godot-automation-nixos.webp"
tags = ["linux", "nixos", "godot"]
keywords = ["linux", "nixos", "godot"]
description = ""
author = "Matthew Croughan"
authorTwitter = "matthewcroughan"
showFullContent = false
+++

# The pain

Reading the [Exporting for
Android](https://docs.godotengine.org/en/3.4/tutorials/export/exporting_for_android.html#exporting-for-android)
documentation for Godot is grim. It instructs developers to manually execute
series of esoteric runes in order to *hopefully* achieve an end result and
environment that is the same as everyone else. It expects you, the user, to
"Ensure that the required packages are installed as well.", at specific
versions. It does not tell you how to go about getting those versions of the
required programs, whilst encouraging you to use `sdkmanager`, a constantly
mutating script which downloads random files and binaries from the internet,
compiling nothing from source, and leaving two developers in very different
states.

After the developer has built their environment using [nothing but a box of
scraps](https://www.youtube.com/watch?v=H4oHU3RXjiM), they're expected to do
even more:

Now, they must:

- Fill in strings by hand via the Godot GUI, which modifies the state of
  the `~/.config/godot/editor_settings-3.tres` file, something unique to their
  system.

- Allow Godot to download files from the internet and place precompiled
  static binaries known as [export templates] from the internet, placing them in
  `~/.local/share/godot/templates/`

- Generate an Android debug keystore, for deploying to a development device.

Each step is required if you want to export your Godot game for Android. If you
miss a step or perform it in the wrong order, you risk hours of debugging. Do
developers really have to go through all of this trouble, just to achieve the
simple goal of exporting for Android? Can't some sort of system handle this for
us?

# Patching Godot to pieces

[Julian Todd](https://github.com/goatchurchprime/), makes a project called
[TunnelVR](https://github.com/tunnelvr/tunnelvr), which is aiming to be like
OpenStreetMaps for caves. He needs Android export capabilities, and I don't want
him to go through the pain explained above. 

I want him to be able to run a single command in order to obtain an environment
in which everything is present and already wired into Godot, this means:

- The Android SDK, without needing to run a series of random `sdkmanager`
   commands.
- The Godot Export Templates, without needing to click any
   buttons in Godot.
- A debug android keystore, without needing to manually generate keys using the
   `keytool` command.

If only there were a way to tell Godot where to find the Android SDK, Godot
Export Templates and Android keystore, perhaps via the cli or by an environment
variable; bypassing their `editor_settings-3.tres` config file which is
constantly being overwritten as you fill in boxes in the editor. That way, I
could grab those in a way that I control, constructing an environment for a
developer in my own way, simply telling Godot where to find these files.

I got tired of [*waiting for
Godot*](https://en.wikipedia.org/wiki/Waiting_for_Godot), so patched the source
code using [Nix](https://nixos.org/guides/install-nix.html), to accomplish
exactly that, and was very successful. My commits and pull request are visible
[here](https://github.com/tunnelvr/tunnelvr/pull/28), let's take a look at what
I did.

## Defining a patched Godot

We need to make Godot read environment variables for the paths we want to
control. We can only do this by modifying Godot's source code, but they're
unlikely to accept a patch for this functionality.

I define `my-godot`. It is equal to the `godot` we already have from Nixpkgs,
except we're going to make a few changes. We use `overrideAttrs`, a function
that lives inside of all packages in Nixpkgs. This function lets us say "I want
Godot, as already described by Nixpkgs, except I want to change the source code
used, and I wanna patch some of the code, and I want to change the version
string, etc...".

```nix
...
  my-godot = godot.overrideAttrs (oldAttrs: rec {
    version = godot-source.rev;
    src = godot-source;
    preBuild =
      ''
        substituteInPlace platform/android/export/export_plugin.cpp \
          --replace 'String sdk_path = EditorSettings::get_singleton()->get("export/android/android_sdk_path")' 'String sdk_path = std::getenv("tunnelvr_ANDROID_SDK")'
        substituteInPlace platform/android/export/export_plugin.cpp \
          --replace 'EditorSettings::get_singleton()->get("export/android/debug_keystore")' 'std::getenv("tunnelvr_DEBUG_KEY")'
        substituteInPlace editor/editor_settings.cpp \
          --replace 'get_data_dir().plus_file("templates")' 'std::getenv("tunnelvr_EXPORT_TEMPLATES")'
      '';
  });
...
```

`substituteInPlace` is roughly equivalent to `sed`, except all I have to say is
"Take this file and replace this string with another string.". In this example,
I am taking the Godot C++ source code and replacing the offending code in the
relevant files with my own.

`EditorSettings::get_singleton()->get("export/android/android_sdk_path` means
that Godot will read my `~/.config/godot/editor_settings-3.tres` to find the
value of the `android_sdk_path` string that the Godot devs assume has been added
by the user. Instead, I'm going to make it equal to an environment variable I
define, `$tunnelvr_ANDROID_SDK`. This means I can now set the path to the
Android SDK in any way of my choosing. I then do the same for every other path I
want to control, by replacing everything with `std::getenv`, rather than their
`EditorSettings` functions, or hard coded paths.

The `preBuild` phase is where I'm choosing to do the patching. This means that
prior to building the source code for Godot, our version of Godot (`my-godot`)
is going to run the shell script `substituteInPlace` on some source code files,
to replace strings that I want to have control over in the Godot source code.
`substituteInPlace` is a shell function that's been gifted to us by Nix, since
Godot is a derivation (nix package) that was made using the
`stdenv.mkDerivation` function that almost every package in Nixpkgs is made
from. It just has this helper function inside of it, ready for us to make use
of.

## Defining a wrapped version of our patched Godot

Now that we've made a version of Godot that will read environment variables to
find our files, we can derive yet another Godot where all those environment
variables are set before you run it. Here, I call it `my-godot-wrapped`.

```nix
...
  my-godot-wrapped = symlinkJoin {
    name = "my-godot-with-android-sdk";
    nativeBuildInputs = [ final.makeWrapper ];
    paths = [ final.my-godot ];
    postBuild =
      let
        # Godot's source code has `version.py` in it, which means we
        # can parse it using regex in order to construct the link to
        # download the export templates from.
        version = rec {
          # Fully constructed string, example: "3.4".
          string = "${major + "." + minor + (final.lib.optionalString (patch != "") "." + patch)}";
          file = "${godot-source}/version.py";
          major = toString (builtins.match ".+major = ([0-9]+).+" (builtins.readFile file));
          minor = toString (builtins.match ".+minor = ([0-9]+).+" (builtins.readFile file));
          patch = toString (builtins.match ".+patch = ([1-9]+).+" (builtins.readFile file));
          # stable, rc, dev, etc.
          status = toString (builtins.match ".+status = \"([A-z]+)\".+" (builtins.readFile file));
        };
        debugKey = final.runCommand "debugKey" {} ''
          ${final.jre_minimal}/bin/keytool -keyalg RSA -genkeypair -alias androiddebugkey -keypass android -keystore debug.keystore -storepass android -dname "CN=Android Debug,O=Android,C=US" -validity 9999 -deststoretype pkcs12
          mv debug.keystore $out
        '';
        export-templates = final.fetchzip {
          url = "https://downloads.tuxfamily.org/godotengine/${version.string}/Godot_v${version.string}-${version.status}_export_templates.tpz";
          sha256 = "sha256-3trC1ocgIVNWN19k6LUnZ6NhDTme+aT7RVL2XmkXzr0=";
          # postFetch is necessary because the downloaded file has a
          # .tpz extension, meaning `fetchzip` cannot otherwise extract
          # it properly. Additionally, the game engine expects the
          # template path to be in a folder by the name of the current
          # version + status, like '3.4-stable/templates' for example,
          # so we accomplish that here.
          postFetch = ''
            unzip $downloadedFile -d ./
            mkdir -p $out/templates/${version.string}.${version.status}
            mv ./templates/* $out/templates/${version.string}.${version.status}
          '';
        };
      in
      ''
        wrapProgram $out/bin/godot \
          --set tunnelvr_ANDROID_SDK "${final.androidenv.androidPkgs_9_0.androidsdk}/libexec/android-sdk"\
          --set tunnelvr_EXPORT_TEMPLATES "${export-templates}/templates" \
          --set tunnelvr_DEBUG_KEY "${debugKey}"
      '';
  };
...

```

This code was a lot of fun to make.

`debugKey` is a simple derivation which uses `keytool` to generate a key, which
is placed in the Nix Store. We use `wrapProgram` to set the `tunnelvr_DEBUG_KEY`
variable to the result of the nix expression `${debugKey}`. That results in a
string like `/nix/store/1pw8k0vl0miqv7kjxkyfk1qq5rywb4rs-debugKey`, which Godot
will now use thanks to our patching. `export-templates` is the same way, and our
`tunnelvr_ANDROID_SDK` variable also trivially fetches the android sdk from
`androidenv.androidPkgs_9_0.androidsdk` in Nixpkgs.

`version` is an attribute set which contains the major, minor, patch and status,
which allows me to construct a string like `3.4-stable` by reading and applying
regex to the
[version.py](https://github.com/godotengine/godot/blob/3.4-stable/version.py)
file that exists in the Godot source code. This allows me to fetch the
export-templates in a way that doesn't force me to hardcode the URL to a string.

## Defining the devShell

We now have our solution. If I want to make this as simple as one command, `nix
develop`, I have to define a `devShell` in the `flake.nix` of this repository.

```nix
...
  devShell = forAllSystems (system:
    let pkgs = nixpkgsFor."${system}";
    in pkgs.mkShell {
      buildInputs = with pkgs; [ my-godot-wrapped jre_headless ];
    });
...
```

This defines a shell which contains `my-godot-wrapped` and `jre_headless`. Turns
out that some of the Android SDK commands have an undeclared dependency on Java,
such as `keytool`, which otherwise crash if it is not present.

# The result

Now that I've patched Godot in this fashion, if you're developing TunnelVR, all
you have to do is `nix develop`, and you can export APKs from Godot. No extra
steps. This will work on any variety of Linux distribution, be it Ubuntu,
Alpine, Arch, it doesn't matter as long as you have `nix`.
