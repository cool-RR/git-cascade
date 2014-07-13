#!/usr/bin/env python3

# Copyright 2009-2014 Ram Rachum.
# This program is distributed under the MIT license.

import sys
import io
import os
import subprocess
import shutil
import contextlib
import collections
import tempfile
import shlex

__doc__ = '''\
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

Also, you may use `.` as an alias for the current branch.
'''

VersionInfo = collections.namedtuple('VersionInfo', 'major minor micro release')

__version__ = '0.2'
__version_info__ = VersionInfo(0, 2, 0, 'alpha')

@contextlib.contextmanager
def create_temp_folder():
    '''
    Context manager that creates a temporary folder and deletes it after usage.
    '''
    temp_folder = tempfile.mkdtemp(prefix='git-forward-merge')
    yield temp_folder
    shutil.rmtree(temp_folder)


def run(command, show=True, assert_success=True, env=None):
    popen = subprocess.Popen(
        shlex.split(('bash -c "%s"' % command)), stdout=subprocess.PIPE,
        stderr=subprocess.PIPE, env=env
    )
    stdout, stderr = map(str.strip, map(bytes.decode, popen.communicate()))
    
    if show:
        if stdout: print(stdout)
        if stderr: print(stderr)
    if popen.returncode != 0:
        print('Failure')
        raise SystemExit(1)
    return stdout


def get_fast_forward_status(branch_1, branch_2):
    first_commit = run('git merge-base %s %s' % (branch_1, branch_1),
                       show=False)
    second_commit = run('git merge-base %s %s' % (branch_2, branch_2),
                        show=False)
    base_commit=run('git merge-base %s %s' % (branch_1, branch_2), show=False)
    if second_commit == base_commit:
        return 1
    elif first_commit == base_commit:
        return -1
    else:
        return 0

def show_help_and_exit():
    print(__doc__)
    raise SystemExit(0)
    

git_top_level = run('git rev-parse --show-toplevel', show=False)

if '--help' in sys.argv:
    show_help_and_exit()

config_lines = run('git config --list', show=False).split('\n')

###############################################################################
#                                                                             #
alias_strings = [line[19:] for line in config_lines if
                   line.startswith('git-branch-aliases.')]
aliases_dict = dict(
    alias_string.split('=', 1) for alias_string in alias_strings
)
def expand_branch_name(short_branch_name):
    if short_branch_name == '.':
        return current_branch
    else:
        return aliases_dict.get(short_branch_name, short_branch_name)
#                                                                             #
###############################################################################

current_branch = run('git rev-parse --abbrev-ref HEAD')    

### Analyzing input to get source and destinations: ###########################
#                                                                             #
branches = tuple(map(expand_branch_name, map(str.strip, sys.argv[1:])))
if not branches:
    show_help_and_exit()
if len(branches) == 1:
    source = 'HEAD'
    destinations = {branches[0]}
else:
    assert len(branches) >= 2
    source = branches[0]
    destinations = set(branches[1:])
#                                                                             #
### Finished analyzing input to get source and destinations. ##################

print('Pushing %s' % (', '.join(
      (('%s -> %s' % (source, destination)) for destination in destinations))))

for destination in destinations:
    if destination == current_branch:
        print('Since %s is currently checked out, using `git merge` '
              'directly...' % destination)
        run('git merge %s' % source)
    else:
        fast_forward_status = get_fast_forward_status(source, destination)
        if fast_forward_status == 1:
            run('git push . %s:%s' % (source, destination))
        elif fast_forward_status == 0:
            GIT_INDEX_FILE = '%s/.git/aux-merge-index' % git_top_level
            with create_temp_folder() as GIT_WORK_TREE:
                def run_in_sandbox(command):
                    env = os.environ.copy()
                    env.update({'GIT_INDEX_FILE': GIT_INDEX_FILE,
                                'GIT_WORK_TREE': GIT_WORK_TREE})
                    return run(command, env=env)
                try:
                    os.mkdir('%s.git' % GIT_WORK_TREE)
                    run_in_sandbox(
                        "git read-tree -im %s %s %s" %
                            (run_in_sandbox('git merge-base %s %s' %
                                                        (destination, source)),
                             destination, source)
                    )
                    run_in_sandbox('git merge-index git-merge-one-file -a')
                    run_in_sandbox(
                        "git write-tree | xargs -i@ git commit-tree @ -p "
                        "{destination} -p {source} -m 'Merge {source} into "
                        "{destination}' | xargs git update-ref -m'Merge "
                        "{source} into {destination}' "
                        "refs/heads/{destination};"
                                .format(source=source, destination=destination)
                    )
                finally:
                    os.remove(GIT_INDEX_FILE)
        else:
            assert fast_forward_status == -1
            print('%s is already ahead of %s, no need to merge.' %
                                                         (destination, source))

