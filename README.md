Project Tasks
=============

Pt runs utilities specific to a project (or any directory).

    $ cd /path/to/project
    $ readlink .pt
    util
    $ ls util
    UTIL
    $ pt UTIL

The project directory is either the current directory or the closest $PWD directory with a .pt entry.  Utilities may be run from sub-directories.

    $ cd sub/dir
    $ pt UTIL

Without a utility argument, prints the project directory.

    $ pt
    /path/to/project
    $ pwd
    /path/to/project/sub/dir


Quickstart
----------

Install, if ~/bin is in your $PATH:

    $ git clone https://github.com/rdpate/pt.git
    $ ln -s $PWD/pt/pt ~/bin/

Mark your project root:

    $ cd ~/my-project
    $ mkdir util
    $ ln -s util .pt

Create tasks:

    $ cd ~/my-project
    $ cat <<'EXAMPLE' >util/example
    #!/bin/sh -ue
    cat <<END
    Project: $(pt)
    From:    $PWD
    With:    $*
    END
    EXAMPLE
    $ chmod +x util/example
    $ cd sub/dir/in/project
    $ pt example abra cadabra
    Project: /home/user/my-project
    From:    /home/user/my-project/sub/dir/in/project
    With:    abra cadabra

Or without a dedicated sub-directory:

    $ ln -s . .pt

---

*The art of simplicity is a puzzle of complexity.* — Douglas Horton

---

Notes
-----

Executing arbitrary files from $PWD may smell – though this is the same situation as "./configure" or "make whatever".

The search for a parent directory with a ".pt" subdirectory uses $PWD rather than following ".." entries.  If $PWD contains a symlink, the lookup will be different.

While $HOME may have a ".pt", this is strongly discouraged when $HOME corresponds to a normal user account.  Becoming accustomed to user-specific Pt in "~/.pt" rather than project-specific utilities would lead to later executing unexpected files based on $PWD.


Project Shell
-------------

Prepend a project directory to $PATH and start a shell:

    $ cat /path/to/project/util/shell
    #!/bin/sh -ue
    PATH="$(readlink -f "$(dirname "$0")/../cmd"):$PATH"
    exec "$SHELL" "$@"


Generic Use
-----------

These shell functions use Pt without using project-specific utilities.  (A .pt entry must still exist, but it can be an empty directory/file or "ln -s . .pt".)

Change to project root or sub-directory:

    cdp() {
        local project
        project="$(pt)" || return 1
        # allows both "cdp" and "cdp subdir"
        cd "$project/$1"
    }

Start a program with a project-relative file:

    vs() {
        local session
        session="$(pt)/.session.vim" || return 1
        if [ ! -e "$session" ]; then
            printf %s\\n 'let v:this_session = expand("<sfile>:p")' >"$session"
        fi
        vim -S "$session"
    }


Interactions with VCS
---------------------

Mercurial provides "hg root".  Git provides "git rev-parse --show-toplevel".  While neither provides a convenient way to execute "project commands" relative to that location, these "root" lookups could be used instead of Pt.  However, Pt avoids tying a "project" to a repo's root, thus allowing both separation from the VCS tool and use after export from a repository.


Phony Targets
-------------

Make has "phony" targets which are often used for tasks, such as clean, install, build (commonly named "all").  These fake targets can instead be turned into tasks ("pt clean", "pt build") and a "make" task created which only accepts real targets as arguments.

Example setup:

    $ cd /path/to/project
    $ mkdir -p sub/dir
    $ cat Makefile
    sub/dir/file:
            @printf %s\\n "$$(pwd)  $@"
    $ cat .pt/make
    #!/bin/sh -ue
    #
    # for exposition:
    # - assume all arguments are paths to files
    # - assume all paths are relative to PWD
    # - assume no paths use ".."
    # - handle arguments sequentially (ignore make's parallel capability)
    # - depend on Pt, though normally shouldn't

    if [ $# = 0 ]; then
        set -- all
    fi
    project="$(pt)"
    cwd_offset="${PWD#"$project"}"
    if [ -n "$cwd_offset" ]; then
        cwd_offset="${cwd_offset#/}/"
    fi
    cd "$project"
    for filename; do
        make "$cwd_offset$filename"
    done

Note the final make command is the same, from the project root as working directory, no matter where in the project tree you start:

    $ cd /path/to/project
    $ pt make sub/dir/file
    /path/to/project  sub/dir/file
    $ cd sub
    $ pt make dir/file
    /path/to/project  sub/dir/file
    $ cd dir
    $ pt make file
    /path/to/project  sub/dir/file

And then change all existing phony targets into tasks:

    $ cat /path/to/project/util/clean
    #!/bin/sh -ue
    cd "$(dirname "$0")/.."
    find . -name '*.py[co]' -delete
