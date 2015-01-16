git cascade and git forward-merge
=====================================

This package contains the following scripts:

 - `git cascade` - Cascade changes from each branch to its dependents.
 - `git forward-merge` - Merge branches without checking them out.


Requirements
------------

 - Python 3.x. (Which must be accessible using `python3`; if you're on 
Windows you might need to set this up.)


Installation
------------

Installation is manual unfortunately, and basically involves putting both
scripts in a directory that's in your `$PATH`. After downloading/cloning the
repo, follow the instructions according to your operating system:


### Linux ###

Copy the two script files to `/usr/local/bin`. That's it. 


### Windows ###

On Windows with Msysgit, you can copy them into `C:\Program Files (x86)\Git\bin`.

Following caveats:

 - You will probably have to replace the shebang line with `#!c:\python34\python.exe` or wherever else you have Python 3 installed.
 - You can only run these scripts from the Git Bash. If you haven't installed Msysgit with `Git Bash Here` option, invoke `bin\bash.exe`.
 - You have to run `git-cascade` and `git-forward-merge` instead of `git cascade` and `git forward-merge` due to Windows shenanigans.


### Mac ###

Peruse [this guide] (http://shapeshed.com/using_custom_shell_scripts_on_osx_or_linux/) and use it for the two scripts in this repo.



git cascade - Cascade changes from each branch to its dependents.
=================================================================

This command:

    git cascade foo master

Merges the branch `foo` into branch `master`, and into every branch that's
behind `master` (regarding the cascade order, not to confuse with Git's
'ahead'/'behind'-terminology which is referring to tracking branches).
What does that mean?

In many projects, there's a progressive order of branches that changes go
through. For example:

    development > staging > master

This means that a new change might first get committed into development, then
after testing will be merged into `staging`, then after further testing get
merged into `master`.

That's all well and good, but sometimes you want to take a shortcut and push a
change directly into a more advanced branch. (This can happen if the change is
small enough that you're sure it won't break anything, or the change is urgent
and needs to be pushed to production immediately, or a bunch of other reasons.)
In these cases, you want to push to `master`, and automatically push to all
branches that are behind `master` in the cascade order. In our example, if you 
run:

    git cascade foo master

The changes in `foo` will be automatically pushed to `master`, `staging` and
`development`, all in one command, without having to check any of them out.

If you were to type:

    git cascade foo staging

Then the changes will be pushed only to `staging` and `development`, with
master remaining untouched.


Defining cascades
-----------------

To define the cascade order for your project, put it in this format in your
project config: (.git/config)

    [git-cascade]
        cascade = development > staging > master

You may also include multiple lines to define more complex cascade trees:

    [git-cascade]
        cascade = development > staging > master
        cascade = other_development > staging

`git cascade` will do the right thing when cascading.

Run `git cascade --show` to show the list of current cascades.


Alternate forms
---------------

If using three or more arguments, the branch specified in the first argument
will be cascaded into all the other branches, and all their dependents. So this:

    git cascade foo staging whatever

Will cascade `foo` into `staging`, `whatever` and all of their dependents.

If only one argument is specified, git cascade will assume you want to cascade
`HEAD` (the current commit) into the specified branch. So this:

    git cascade staging

Will cascade `HEAD` into `staging` and all of its dependents.

If no arguments are specified:

    git cascade
    
Then HEAD will be cascaded into the current branch. (It's often useful to
cascade a branch into itself because it also merges it into the branches it
flows into.)


How does it work?
-----------------

The merges in `git cascade` are done by `git forward-merge`, which creates a
temporary git index file and working directory to be used only for the merge,
without interfering with the actual index file and working directory. (Unless
the merge is a fast-forward, in which case the merge is done trivially by a
local push.)


Limitation
----------

`git cascade` works only when the merge can be done automatically. It doesn't
work for merges that require conflict resolution. For that, please resort to
using `git merge`.

If you do attempt a cascade that results in a merge that requires conflict
resolution, `git cascade` will abort the merge and leave your working
directory clean, UNLESS the branch you're merging to is the current branch, in
which case it will leave the merge in the working directory for you to resolve
the conflict and commit, just like `git merge`.

(If there were multiple merges, all merges up to the failing one will be
completed in the repo.)


Branch aliases
--------------

You may use branch aliases when using `git cascade`. So,
if you defined something like this in your config, (either the repo-specific
config or the global one):

    [git-branch-aliases]
        s = staging
        m = master

And then run:

    git cascade foo m

It will cascade `foo` into `master` and all of its dependents.


git forward-merge - Merge branches without checking them out.
=============================================================

This command was written in order to solve an annoyance with the built-in `git
merge`. The annoying thing about `git merge` is that if one wants to merge
branch `foo` into branch `bar`, one first needs to check out branch `bar`, and
only then merge `foo` into it. This can become a drag, especially when having
an unclean working tree.

Enter `git forward-merge`. All pushes done with it work regardless of which
branch is currently checked out and which files are in the working tree.

Push branch `foo` into `bar`:

    git forward-merge foo bar

Push current branch/commit into `bar`:

    git forward-merge bar

Push branch `foo` into `bar`, `baz` and `meow`:

    git forward-merge foo bar baz meow


How does it work?
-----------------

`git forward-merge` creates a temporary git index file and working directory to
be used only for the merge, without interfering with the actual index file and
working directory. (Unless the merge is a fast-forward, in which case the merge
is done trivially by a local push.)


Limitation
----------

`git forward-merge` works only when the merge can be done
automatically. It doesn't work for merges that require conflict resolution. For
that, please resort to using `git merge`.

If you do attempt a merge that requires conflict resolution, `git
forward-merge` will abort the merge and leave your working directory clean,
UNLESS the branch you're merging to is the current branch, in which case it
will leave the merge in the working directory for you to resolve the conflict
and commit, just like `git merge`.


Branch aliases
--------------

You may use branch aliases when using `git forward-merge`. So,
if you defined something like this in your config, (either the repo-specific
config or the global one):

    [git-branch-aliases]
        s = staging
        m = master

And then run:

    git forward-merge s m

It will merge `staging` into `master`.


Copyright and license
=====================

Both scripts are copyright 2009-2014 Ram Rachum and are released under the MIT
license. I provide 
[development services in Python and Django](https://chipmunkdev.com).



