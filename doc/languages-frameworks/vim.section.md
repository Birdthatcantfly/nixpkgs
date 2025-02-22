# Vim {#vim}

Both Neovim and Vim can be configured to include your favorite plugins
and additional libraries.

Loading can be deferred; see examples.

At the moment we support three different methods for managing plugins:

- Vim packages (*recommend*)
- VAM (=vim-addon-manager)
- Pathogen
- vim-plug

## Custom configuration {#custom-configuration}

Adding custom .vimrc lines can be done using the following code:

```nix
vim_configurable.customize {
  # `name` optionally specifies the name of the executable and package
  name = "vim-with-plugins";

  vimrcConfig.customRC = ''
    set hidden
  '';
}
```

This configuration is used when Vim is invoked with the command specified as name, in this case `vim-with-plugins`.
You can also omit `name` to customize Vim itself. See the
[definition of `vimUtils.makeCustomizable`](https://github.com/NixOS/nixpkgs/blob/master/pkgs/applications/editors/vim/plugins/vim-utils.nix#L408)
for all supported options.

For Neovim the `configure` argument can be overridden to achieve the same:

```nix
neovim.override {
  configure = {
    customRC = ''
      # here your custom configuration goes!
    '';
  };
}
```

If you want to use `neovim-qt` as a graphical editor, you can configure it by overriding Neovim in an overlay
or passing it an overridden Neovimn:

```nix
neovim-qt.override {
  neovim = neovim.override {
    configure = {
      customRC = ''
        # your custom configuration
      '';
    };
  };
}
```

## Managing plugins with Vim packages {#managing-plugins-with-vim-packages}

To store you plugins in Vim packages (the native Vim plugin manager, see `:help packages`) the following example can be used:

```nix
vim_configurable.customize {
  vimrcConfig.packages.myVimPackage = with pkgs.vimPlugins; {
    # loaded on launch
    start = [ youcompleteme fugitive ];
    # manually loadable by calling `:packadd $plugin-name`
    # however, if a Vim plugin has a dependency that is not explicitly listed in
    # opt that dependency will always be added to start to avoid confusion.
    opt = [ phpCompletion elm-vim ];
    # To automatically load a plugin when opening a filetype, add vimrc lines like:
    # autocmd FileType php :packadd phpCompletion
  };
}
```

`myVimPackage` is an arbitrary name for the generated package. You can choose any name you like.
For Neovim the syntax is:

```nix
neovim.override {
  configure = {
    customRC = ''
      # here your custom configuration goes!
    '';
    packages.myVimPackage = with pkgs.vimPlugins; {
      # see examples below how to use custom packages
      start = [ ];
      # If a Vim plugin has a dependency that is not explicitly listed in
      # opt that dependency will always be added to start to avoid confusion.
      opt = [ ];
    };
  };
}
```

The resulting package can be added to `packageOverrides` in `~/.nixpkgs/config.nix` to make it installable:

```nix
{
  packageOverrides = pkgs: with pkgs; {
    myVim = vim_configurable.customize {
      # `name` specifies the name of the executable and package
      name = "vim-with-plugins";
      # add here code from the example section
    };
    myNeovim = neovim.override {
      configure = {
      # add here code from the example section
      };
    };
  };
}
```

After that you can install your special grafted `myVim` or `myNeovim` packages.

### What if your favourite Vim plugin isn’t already packaged? {#what-if-your-favourite-vim-plugin-isnt-already-packaged}

If one of your favourite plugins isn't packaged, you can package it yourself:

```nix
{ config, pkgs, ... }:

let
  easygrep = pkgs.vimUtils.buildVimPlugin {
    name = "vim-easygrep";
    src = pkgs.fetchFromGitHub {
      owner = "dkprice";
      repo = "vim-easygrep";
      rev = "d0c36a77cc63c22648e792796b1815b44164653a";
      sha256 = "0y2p5mz0d5fhg6n68lhfhl8p4mlwkb82q337c22djs4w5zyzggbc";
    };
  };
in
{
  environment.systemPackages = [
    (
      pkgs.neovim.override {
        configure = {
          packages.myPlugins = with pkgs.vimPlugins; {
          start = [
            vim-go # already packaged plugin
            easygrep # custom package
          ];
          opt = [];
        };
        # ...
      };
     }
    )
  ];
}
```

### Specificities for some plugins
#### Tree sitter

By default `nvim-treesitter` encourages you to download, compile and install
the required tree-sitter grammars at run time with `:TSInstall`. This works
poorly on NixOS.  Instead, to install the `nvim-treesitter` plugins with a set
of precompiled grammars, you can use `nvim-treesitter.withPlugins` function:

```nix
(pkgs.neovim.override {
  configure = {
    packages.myPlugins = with pkgs.vimPlugins; {
      start = [
        (nvim-treesitter.withPlugins (
          plugins: with plugins; [
            tree-sitter-nix
            tree-sitter-python
          ]
        ))
      ];
    };
  };
})
```

To enable all grammars packaged in nixpkgs, use `(pkgs.vimPlugins.nvim-treesitter.withPlugins (plugins: pkgs.tree-sitter.allGrammars))`.

## Managing plugins with vim-plug {#managing-plugins-with-vim-plug}

To use [vim-plug](https://github.com/junegunn/vim-plug) to manage your Vim
plugins the following example can be used:

```nix
vim_configurable.customize {
  vimrcConfig.packages.myVimPackage = with pkgs.vimPlugins; {
    # loaded on launch
    plug.plugins = [ youcompleteme fugitive phpCompletion elm-vim ];
  };
}
```

For Neovim the syntax is:

```nix
neovim.override {
  configure = {
    customRC = ''
      # here your custom configuration goes!
    '';
    plug.plugins = with pkgs.vimPlugins; [
      vim-go
    ];
  };
}
```

## Managing plugins with VAM {#managing-plugins-with-vam}

### Handling dependencies of Vim plugins {#handling-dependencies-of-vim-plugins}

VAM introduced .json files supporting dependencies without versioning
assuming that "using latest version" is ok most of the time.

### Example {#example}

First create a vim-scripts file having one plugin name per line. Example:

```vim
"tlib"
{'name': 'vim-addon-sql'}
{'filetype_regex': '\%(vim)$', 'names': ['reload', 'vim-dev-plugin']}
```

Such vim-scripts file can be read by VAM as well like this:

```vim
call vam#Scripts(expand('~/.vim-scripts'), {})
```

Create a default.nix file:

```nix
{ nixpkgs ? import <nixpkgs> {}, compiler ? "ghc7102" }:
nixpkgs.vim_configurable.customize { name = "vim"; vimrcConfig.vam.pluginDictionaries = [ "vim-addon-vim2nix" ]; }
```

Create a generate.vim file:

```vim
ActivateAddons vim-addon-vim2nix
let vim_scripts = "vim-scripts"
call nix#ExportPluginsForNix({
\  'path_to_nixpkgs': eval('{"'.substitute(substitute(substitute($NIX_PATH, ':', ',', 'g'), '=',':', 'g'), '\([:,]\)', '"\1"',"g").'"}')["nixpkgs"],
\  'cache_file': '/tmp/vim2nix-cache',
\  'try_catch': 0,
\  'plugin_dictionaries': ["vim-addon-manager"]+map(readfile(vim_scripts), 'eval(v:val)')
\ })
```

Then run

```bash
nix-shell -p vimUtils.vim_with_vim2nix --command "vim -c 'source generate.vim'"
```

You should get a Vim buffer with the nix derivations (output1) and vam.pluginDictionaries (output2).
You can add your Vim to your system's configuration file like this and start it by "vim-my":

```nix
my-vim =
  let plugins = let inherit (vimUtils) buildVimPluginFrom2Nix; in {
    copy paste output1 here
  }; in vim_configurable.customize {
    name = "vim-my";

    vimrcConfig.vam.knownPlugins = plugins; # optional
    vimrcConfig.vam.pluginDictionaries = [
       copy paste output2 here
    ];

  };
```

Sample output1:

```nix
"reload" = buildVimPluginFrom2Nix { # created by nix#NixDerivation
  name = "reload";
  src = fetchgit {
    url = "https://github.com/xolox/vim-reload";
    rev = "0a601a668727f5b675cb1ddc19f6861f3f7ab9e1";
    sha256 = "0vb832l9yxj919f5hfg6qj6bn9ni57gnjd3bj7zpq7d4iv2s4wdh";
  };
  dependencies = ["nim-misc"];

};
[...]
```

Sample output2:

```nix
[
  ''vim-addon-manager''
  ''tlib''
  { "name" = ''vim-addon-sql''; }
  { "filetype_regex" = ''\%(vim)$$''; "names" = [ ''reload'' ''vim-dev-plugin'' ]; }
]
```

## Adding new plugins to nixpkgs {#adding-new-plugins-to-nixpkgs}

Nix expressions for Vim plugins are stored in [pkgs/applications/editors/vim/plugins](https://github.com/NixOS/nixpkgs/tree/master/pkgs/applications/editors/vim/plugins). For the vast majority of plugins, Nix expressions are automatically generated by running [`./update.py`](https://github.com/NixOS/nixpkgs/blob/master/pkgs/applications/editors/vim/plugins/update.py). This creates a [generated.nix](https://github.com/NixOS/nixpkgs/blob/master/pkgs/applications/editors/vim/plugins/generated.nix) file based on the plugins listed in [vim-plugin-names](https://github.com/NixOS/nixpkgs/blob/master/pkgs/applications/editors/vim/plugins/vim-plugin-names). Plugins are listed in alphabetical order in `vim-plugin-names` using the format `[github username]/[repository]@[gitref]`. For example https://github.com/scrooloose/nerdtree becomes `scrooloose/nerdtree`.

Some plugins require overrides in order to function properly. Overrides are placed in [overrides.nix](https://github.com/NixOS/nixpkgs/blob/master/pkgs/applications/editors/vim/plugins/overrides.nix). Overrides are most often required when a plugin requires some dependencies, or extra steps are required during the build process. For example `deoplete-fish` requires both `deoplete-nvim` and `vim-fish`, and so the following override was added:

```nix
deoplete-fish = super.deoplete-fish.overrideAttrs(old: {
  dependencies = with super; [ deoplete-nvim vim-fish ];
});
```

Sometimes plugins require an override that must be changed when the plugin is updated. This can cause issues when Vim plugins are auto-updated but the associated override isn't updated. For these plugins, the override should be written so that it specifies all information required to install the plugin, and running `./update.py` doesn't change the derivation for the plugin. Manually updating the override is required to update these types of plugins. An example of such a plugin is `LanguageClient-neovim`.

To add a new plugin, run `./update.py --add "[owner]/[name]"`. **NOTE**: This script automatically commits to your git repository. Be sure to check out a fresh branch before running.

Finally, there are some plugins that are also packaged in nodePackages because they have Javascript-related build steps, such as running webpack. Those plugins are not listed in `vim-plugin-names` or managed by `update.py` at all, and are included separately in `overrides.nix`. Currently, all these plugins are related to the `coc.nvim` ecosystem of Language Server Protocol integration with vim/neovim.

## Updating plugins in nixpkgs {#updating-plugins-in-nixpkgs}

Run the update script with a GitHub API token that has at least `public_repo` access. Running the script without the token is likely to result in rate-limiting (429 errors). For steps on creating an API token, please refer to [GitHub's token documentation](https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token).

```sh
GITHUB_API_TOKEN=my_token ./pkgs/applications/editors/vim/plugins/update.py
```

Alternatively, set the number of processes to a lower count to avoid rate-limiting.

```sh
./pkgs/applications/editors/vim/plugins/update.py --proc 1
```

## Important repositories {#important-repositories}

- [vim-pi](https://bitbucket.org/vimcommunity/vim-pi) is a plugin repository
  from VAM plugin manager meant to be used by others as well used by

- [vim2nix](https://github.com/MarcWeber/vim-addon-vim2nix) which generates the
  .nix code
