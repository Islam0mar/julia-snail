# Julia Snail

![img](snail.png)

Snail is a development environment and REPL interaction package for Julia in the spirit of Common Lisp’s [SLIME](https://common-lisp.net/project/slime/) and Clojure’s [CIDER](https://cider.mx). It enables convenient and dynamic REPL-driven development.


## Features

- **REPL display:** Snail uses [libvterm](https://github.com/neovim/libvterm) with [Emacs bindings](https://github.com/akermu/emacs-libvterm) to display Julia’s native REPL in a good terminal emulator. As a result, the REPL has good performance and far fewer display glitches than attempting to run the REPL in an Emacs-native `term.el` buffer.
- **REPL interaction:** Snail provides a bridge between Julia code and a Julia process running in a REPL. The bridge allows Emacs to interact with and introspect the Julia image. Among other things, this allows loading entire files and individual functions into running Julia processes.
- **Cross-referencing:** Snail is integrated with the built-in Emacs [xref](https://www.gnu.org/software/emacs/manual/html_node/emacs/Xref.html) system. When a Snail session is active, it supports jumping to definitions of functions and macros loaded in the session.
- **Completion:** Snail is also integrated with the built-in Emacs [completion-at-point](https://www.gnu.org/software/emacs/manual/html_node/elisp/Completion-in-Buffers.html) facility. Provided it is configured with the `company-capf` backend, [company-mode](http://company-mode.github.io/) completion will also work (this should be the case by default).
- **Parser:** Snail contains a limited but serviceable Julia parser, used to infer the structure of source files and to enable features which require an understanding of code context, especially the module in which a particular piece of code lives. This enables awareness of the current module for completion and cross-referencing purposes.


## Demo

![img](https://github.com/gcv/julia-snail/wiki/screencasts/screencast-2020-01-26.gif)


## Installation

Julia versions 1.0–1.3 work. No packages need to be installed on the Julia side (other than Julia itself).

On the Emacs side:

1. Make sure you have Emacs 26.2 or later, compiled with module support (`--with-modules`). Check the value of `module-file-suffix`: it should be non-nil. (This is currently a default compile-time option Emacs distributed with [Homebrew](https://formulae.brew.sh/formula/emacs).)
2. Install [libvterm](https://github.com/neovim/libvterm). It is available in [Homebrew](https://formulae.brew.sh/formula/libvterm) and [Ubuntu 19.10](https://packages.ubuntu.com/eoan/libvterm-dev), and in source form on other systems.
3. Install [emacs-libvterm](https://github.com/akermu/emacs-libvterm) using your Emacs package manager. It is available from [MELPA](https://melpa.org/#/vterm) as `vterm`, so use something like `(package-install 'vterm)` or `(use-package vterm)`. **It is important to do this step separately from the `julia-snail` installation, as you may run into problems with the Emacs package manager and byte-compiler!**
4. Verify that `vterm` works by running `M-x vterm` to start a shell. It should display a nice terminal buffer. You may find it useful to customize and configure `vterm`.
5. Install `julia-snail` using your Emacs package manager (see below for a sample `use-package` invocation). It is available on [MELPA](https://melpa.org/#/julia-snail) and [MELPA Stable](https://stable.melpa.org/#/julia-snail).


## Configuration

### `use-package` setup

**Make sure to install vterm first!** (See the [Installation](#installation) section.)

```elisp
(use-package julia-snail
  :hook (julia-mode . julia-snail-mode)
  :config (progn
            ;; order matters, unfortunately:
            (add-to-list 'display-buffer-alist
                         ;; match buffers named "*julia" in general
                         '("\\*julia"
                           ;; actions:
                           (display-buffer-reuse-window display-buffer-same-window)))
            (add-to-list 'display-buffer-alist
                         ;; when displaying buffers named "*julia" in REPL mode
                         '((lambda (bufname _action)
                             (and (string-match-p "\\*julia" bufname)
                                  (with-current-buffer bufname
                                    (bound-and-true-p julia-snail-repl-mode))))
                           ;; actions:
                           (display-buffer-reuse-window display-buffer-pop-up-window)))
            ))
```


### Manual setup

```elisp
(add-to-list 'load-path "/path/to/julia-snail")
(require 'julia-snail)
(add-hook 'julia-mode-hook #'julia-snail-mode)
```

- Configure `display-buffer-alist` to make REPL window switching smoother.
- Configure key bindings in `julia-snail-mode-map` as desired.


### `display-buffer-alist` notes

Window and buffer display behavior is one of the worst defaults Emacs ships with. [Please refer to remarks on the subject written elsewhere](https://github.com/nex3/perspective-el/#some-musings-on-emacs-window-layouts). Some packages go to great lengths to provide clean custom window management (e.g., Magit), but Snail does not have the resources to implement such elaborate schemes.

The configuration above makes various `display-buffer` operations Snail uses have more-or-less sane behavior which disturbs the user’s window layout as little as possible.

It is likely that most users will want the default REPL pop-up behavior to split the window vertically, however the default `split-window-sensibly` implementation only splits that way if `split-height-threshold` is smaller than the current window height. `split-height-threshold` defaults to 80 (lines), and relatively few windows will be that tall. Therefore, consider adding something like the following to your configuration:

```elisp
(setq split-height-threshold 15)
```


## Usage

### Basics

Once Snail is properly installed, open a Julia source file. If `julia-mode-hook` has been correctly configured, `julia-snail-mode` should be enabled in the buffer (look for the Snail lighter in the modeline).

Start a Julia REPL using `M-x julia-snail` or `C-c C-z`. This will load all the Julia-side supporting code Snail requires, and start a server. The server runs on a TCP port (10011 by default) on localhost. You will see `JuliaSnail.start(<port>)` execute on the REPL.

The REPL buffer uses `libvterm` mode, and `libvterm` configuration and key bindings will affect it.

If the Julia program uses Pkg, then run `M-x julia-snail-package-activate` or `C-c C-a` to enable it. (Doing this using REPL commands like `]` also works as normal.)

Load the current Julia source file using `M-x julia-snail-send-buffer-file` or `C-c C-k`. Notice that the REPL does not show an `include()` call, because the command executed across the Snail network connection. Among other advantages, this minimizes REPL history clutter.

Once some Julia code has been loaded into the running image, Snail can begin introspecting it for purposes of cross-references and identifier completion.

The `julia-snail-mode` minor mode provides a key binding map (`julia-snail-mode-map`) with the following commands:

| key     | command                         | description                                              |
|---------|---------------------------------|----------------------------------------------------------|
| C-c C-z | julia-snail                     | start a REPL; flip between REPL and source               |
| C-c C-a | julia-snail-package-activate    | activate the project using `Project.toml`                |
| C-c C-d | julia-snail-doc-lookup          | display the docstring of the identifier at point         |
| C-c C-c | julia-snail-send-top-level-form | evaluate `end`-terminated block around the point in the current module |
| C-M-x   | julia-snail-send-top-level-form | ditto                                                    |
| C-c C-r | julia-snail-send-region         | evaluate active region in the current module             |
| C-c C-l | julia-snail-send-line           | copy current line directly to REPL                       |
| C-c C-k | julia-snail-send-buffer-file    | `include()` the current buffer’s file                    |

Several commands include the note “in the current module”. This means the Snail parser will determine the enclosing `module...end` statements, and run the relevant code in that module. If the module has already been loaded, this means its global variables and functions will be available.

In addition, most `xref` commands are available (except `xref-find-references`). `xref-find-definitions`, by default bound to `M-.`, does a decent job of jumping to function and macro definitions. Cross-reference commands are current-module aware.

Completion also works. Emacs built-in completion features, as well as `company-complete`, will do a reasonable job of finding the right completions in the context of the current module (though will not pick up local variables). Completion is current-module aware.


### Multiple REPLs

To use multiple REPLs, set the local variables `julia-snail-repl-buffer` and `julia-snail-port`. They must be distinct per-project. They can be set at the [file level](https://www.gnu.org/software/emacs/manual/html_node/emacs/Specifying-File-Variables.html), or at the [directory level](https://www.gnu.org/software/emacs/manual/html_node/emacs/Directory-Variables.html). The latter approach is recommended, using a `.dir-locals.el` file at the root of a project directory.

For example, consider two projects: `Mars` and `Venus`, both of which you wish to work on at the same time. They live in different directories.

The `Mars` project directory contains the following `.dir-locals.el` file:

```elisp
((julia-mode . ((julia-snail-port . 10050)
                (julia-snail-repl-buffer . "*julia Mars*"))))
```

The `Venus` project directory contains the following `.dir-locals.el` file:

```elisp
((julia-mode . ((julia-snail-port . 10060)
                (julia-snail-repl-buffer . "*julia Venus*"))))
```

(Be sure to refresh any buffers currently visiting files in `Mars` and `Venus` using `find-alternate-file` or similar after changing these variables.)

Now, source files in `Mars` will interact with the REPL running in the `*julia Mars*` buffer, and source files in `Venus` will interact with the REPL running in the `*julia Venus*` buffer.


### Multiple Julia versions

The `julia-snail-executable` variable can be set at the file level or at the directory level and point to different versions of Julia for different projects. It should be a string referencing the executable binary path.

NB: On a Mac, the Julia binary is typically `Contents/Resources/julia/bin/julia` inside the distribution app bundle. You must either make sure `julia-snail-executable` is set to an absolute path, or configure your Emacs `exec-path` to correctly find the `julia` binary.


## Future improvements

### Foundational

- The Julia interaction side of the Snail server is single-threaded (using `@async`). This means the interaction locks up while the REPL is working or running code. Unfortunately, Julia as of version 1.2 does not have user-accessible low-level multithreading primitives necessary to implement a truly multi-threaded Snail server.


### Structural

-  The `libvterm` dependency forces the use of very recent Emacs releases, forces Emacs to be build with module support, complicates support for Windows, and is generally quite gnarly. It would be much better to re-implement the REPL in Elisp.
- The current parser leaves much to be desired. It is woefully incomplete: among many other things, it cannot detect one-line top-level definitions (such as `f(x) = 10x`). In addition: it is slow, and not particularly straightforward in implementation. A rewrite would work better and enable more features. Unfortunately, parsers are hard. :)


### Functional

- Completion does not pick up local variables. This is yet another weakness of the parser.
- A real eldoc implementation would be great, but difficult to do with Julia’s generic functions. The parser would also have to improve (notice a theme here?).
- A debugger would be great.
