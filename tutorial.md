# `git-annex` tutorial: sharing data and code in the Lab and outside

In this tutorial we introduce
[`git-annex`](https://git-annex.branchable.com) and its use within a
research laboratory to help share data and code among lab members,
external collaborators and anonymous users. `git-annex` is a software
tool that extends the more famous software
[`git`](https://git-scm.com/) in convenient ways when dealing with
large files and large repositories. In the following, we introduce
some basic concepts and then describe the scenario and the workflow
that we implemented in our Lab which, we believe, can be useful to
other people in a similar setting. `git` and `git-annex` require some
dedication before reaching fruitful use. During that process, it is
common to make mistakes, as it was for us. For this reason, in this
tutorial, we also describe common errors and how to recover from
them. Notice that, basic familiarity with `git` is assumed as a
pre-requisite for this tutorial.


## The Lab scenario

In our lab, we have large datasets - terabytes of data - which also
comprise many large files and code. Such datasets are kept in a
storage server. Portions of the data are frequently needed on local
desktop and laptop computers of lab members and collaborators, for
processing and analysis. Moreover, new data are frequently generated
by lab members, on local computers, by further processing of available
data. One main aim is to share such new data and the code with
others. Additionally, the data already shared are not static: from
time to time, code is updated or bugs are fixed, so some of the
preprocessed data is re-generated and shared, substituting the
previous version. In such a setting, is is important for lab members
and collaborators to get updates of data and code in a simple way.


## What is `git-annex`?

Simply put, [`git-annex`](https://git-annex.branchable.com) is an
extension of `git` that provides some extra functionalities:
* Large files in the repository are not locally copied, when cloning
  or fetching/pulling. Of course, they can be retrieved on
  request. Additionally, local copies of large files can be removed to
  free some space.
* `git-annex` keeps track of how many and where copies of each file
  are.
* TODO
  
  

### Alternatives to `git-annex`
TODO
* [`git-lfs`](https://git-lfs.github.com/)
* [`datalad`](http://datalad.org/)



## Setting-up a centralized repository
In the following example, we create a directory `/labdata` on a
storage-server, where we store a copy of all the data with `git` and
`git-annex`, to be shared with lab members and external
collaborators. The repository hosts both the `git` database, in
`/labdata/.git/`, and a copy of the actual files and directories, the
*working tree*, in `/labdata/`, for easy browsing.

Permission to add or modify the data in the repository is enforced
through filesystem permissions by creating a group of users, named
`dataowners`. Everyone else can (only) read the data in the
repository.

Here we describe the step-by-step procedure to create the repository
from scratch, with example commands followed by their detailed
explanation:

    cd /
    mkdir labdata
	addgrup dataowners
	chgrp dataowners labdata
	chmod g+rwx labdata
	chmod o+rx-w
	chmod g+s labdata
	cd repository
	
This first group of commands creates the directory to host the
repository `/labdata`, creates a new system group `dataowners` and
sets such group to `/labdata`, with write permissions. Additionally,
read (`r`), write (`w`) and access (`x`) permissions are granted to
the group (`g+rwx`) and read and access (but not write) permissions
are granted to everyone else (`o+rx-w`). Finally, the [`setgid`
permission](https://en.wikipedia.org/wiki/Setuid#setuid_and_setgid_on_directories)
is enabled for the group (`g+s`), so that all future files and
directories created inside `/labdata` will automatically inherit the
group `dataowners` and the `setgid` bit.
	
	git init --shared=group
	git annex init storage-server

This second group of commands creates the `git` repository and the
additional `git-annex` part of it. Notice that, the `git-annex` part
of the repository can only be initialized within an existing `git`
repository. In order to let the repository be group-writable and
accessible to everyone, the initialization of the `git` repository
requires [`--shared=group`](https://git-scm.com/docs/git-init). This
will properly set permissions within `/labdata/.git/`. The
initialization of `git-annex` creates a `/labdata/.git/annex/`
directory, called the *annex*, where `git-annex` stores all its
information. To conclude, we added the *optional* `storage-server`
description when initializing the `git-annex` part of the
repository. This is convenient to set a desired human-readable label
to the repository.

At this point, content/changes can be added to the repository in two
main ways:

* Directly on the storage-server, by copying files and directories in
  `/labdata` and then:
  * either via `git annex add <file>` and `git commit -m
  <message>`. In this case, The file is added to the annex, i.e. moved
  to `/labdata/.git/annex/objects/`, set read-only, renamed according
  to its checksum and a symbolic link pointing to it is created in the
  original location of the file. Only the symbolic link is added to
  the `git` git repository, while `git-annex` keeps track of the
  content. From the user perspective, the initial file is still
  accessible, through the link, in read-only mode. Notice that, when
  cloning this repository, only the symbolic link of this file will be
  present and not its content, unless explicitly requested.
  * Or via `git add <file>` and `git commit -m <message>`. In this
    case the file is added to the `git` repository and *not* to the
    annex. Notice that, when cloning this repository, a copy of this
    file will be present, as always with `git`.
* From remote repositories, through `git push` or `git annex sync`. In
  this second case, the repository must be configured properly, as
  explained below.
  
Using `git annex add <file>` instead of `git add <file>` can be
decided for each file, individually, and depends on the purpose of the
file and of the repository. Typically, code should be added via `git
add <file>` and data via `git annex add <file>`. Nevertheless, it is
possible to use `git annex add <file>` for everything. If, at a later
stage, a file needs to be moved from the `git` repository to the
annex, or viceveresa, 

Here follows an example transcript of what happens when executing `git
annex add <file>` on a file `foo` present in the repository:
  
    > ls -al
	total 16
	drwxrwsr-x  3 ele  dataowners 4096 dic 26 16:19 .
	drwxr-xr-x 26 root root       4096 dic 26 16:13 ..
	-rw-rw-r--  1 ele  dataowners    4 dic 26 16:19 foo
	drwxrwsr-x  9 ele  dataowners 4096 dic 26 16:18 .git
	> git annex add foo
	> ls -al
	total 16
    drwxrwsr-x  3 ele  dataowners 4096 dic 26 16:21 .
    drwxr-xr-x 26 root root       4096 dic 26 16:13 ..
    lrwxrwxrwx  1 ele  dataowners  178 dic 26 16:19 foo -> .git/annex/objects/g7/9v/SHA256E-s4--7d865e959b2466918c9863afca942d0fb89d7c9ac0c99bafc3749504ded97730/SHA256E-s4--7d865e959b2466918c9863afca942d0fb89d7c9ac0c99bafc3749504ded97730
    drwxrwsr-x  9 ele  dataowners 4096 dic 26 16:21 .git
    > git commit -m "added foo"
    [master (root-commit) 3e461c6] added foo
     1 file changed, 1 insertion(+)
     create mode 120000 foo
    > ls -al .git/annex/objects/g7/9v/SHA256E-s4--7d865e959b2466918c9863afca942d0fb89d7c9ac0c99bafc3749504ded97730/SHA256E-s4--7d865e959b2466918c9863afca942d0fb89d7c9ac0c99bafc3749504ded97730
    -rw-rw-rw- 1 ele dataowners 4 dic 26 16:19 .git/annex/objects/g7/9v/SHA256E-s4--7d865e959b2466918c9863afca942d0fb89d7c9ac0c99bafc3749504ded97730/SHA256E-s4--7d865e959b2466918c9863afca942d0fb89d7c9ac0c99bafc3749504ded97730
	


### Allowing content creation from remote repositories
Content and changes can be created on remote clones of the repository,
i.e. local computers of lab members and collaborators. Such contents
and changes need to be pushed to the storage-server, in order to be
shared. For this reasons, the storage-server needs to be properly
configured in order to allow that, in two steps. The first is:

    git config receive.denyCurrentBranch updateInstead
	
With this command, we allow remote users to *push* to the
repository. Normally, this is not permitted, because the repository is
*non bare*, i.e.  it has a working tree of files and directories,
besides the `.git/` database. If you do not plan to push changes from
remote, then you do not need this configuration. The second step is:
	
	cd /labdata
    git annex wanted . standard
	git annex group . backup

The storage-server is meant to keep copies of all files in the
repository. When content is created remotely, it is very important to
tell the storage-server to enforce this desideratum, when operations
like `git annex sync` are performed (see TODO). In order to enforce
such behavior and other similar ones, `git annex` provides rich
expressions to be set, see
https://git-annex.branchable.com/preferred_content/ . But it also
offers standard groups of preferences, as described in
https://git-annex.branchable.com/preferred_content/standard_groups/
. The commands above tells to all the repository to use a standard
group of preferences called *backup*, which means "All content is
wanted. Even content of old/deleted files."


### Adding public accessibility from the web

Information on how to access the repository when the storage server
directory with the data is exposed via web server. Basically:

    cd /labdata
	mv .git/hooks/post-update.sample .git/hooks/post-update

TODO: here we miss some information about how to tailor remotes like:

    git annex initremote datasrc type=git location=HTTPURL/.git autoenable=true


### Publishing the repository on `github.com`
TODO 

The idea is to keep a copy of the repository on github, without the
contents of the annex, so that it is more visible and can be easily
cloned by anonymous users. Moreover, it can be set up so that content
can be retrieved via `git annex get <file>` leveraging the access to
the storage-server and/or the public access for the web.


### Moving content from `git-annex` to `git`
After populating and using the repository, it is common to realize
that it may not be smart to have *all* files stored with `git-annex`
and that is would be better to have them simply stored in `git`. The
following commands migrate files from `git-annex` to `git`:

    git unannex <file>
	git add <file>
	git come -m <message>

Notice that `git unannex <file>` does not need a commit.


#### Problems with permissions when pushing/syncing
TODO



## `git-annex` for data creators

    git clone creator1@storage-server:/labdata
	git annex init creator1-desktop
	# set the repository to not retrieve additional content when syncing:
	git annex wanted . standard
	git annex group . manual



### Editing files
If a `<file>` is stored with `git annex` and changes to it needs to be
made, it must be unlocked first, before making the changes:

    git annex unlock <file>
	edit...edit...edit
	git annex add <file>
	git commit -m "updated <file>"

`git annex unlock <file>` removes the symbolic link and copies the
content of the file in its place, with write permission. Notice that
this is a second copy of the file. After changing the file, `git-annex
add` and `git commit` can be performed as usual. Notice that, if you
need to frequently change a file, it may be more convenient to store
it with `git` instead of `git-annex`.


#### Issue: Changing a file without unlocking first
What happens if you attempt to edit a file without unlocking first?
Files added with `git-annex` appears as symbolic links in the
filesystem. An application, such as an editor, should warn that you
are opening a link and not a file. Secondly, the content of the file,
pointed by the link, is stored in `.git/annex/objects/` and set as
write-protected. This is the only copy of the content of the file in
the local repository, that is why it is protected. The application
attempting to write on this file should either fail, with
`permission denied`, or clearly ask confirmation to write on a write
protected file, e.g. Sublime Text 3. If the user insists to write on
the file and the application allows that, basically the internal copy
of `git-annex` is damaged. With `git annex fsck <file>`, `git-annex`
will tell first that the local copy of the file is not good anymore
and will put it in `.git/annex/bad/`. In order to solve such a
situation, it is necessary to retrieve a pristine copy of the file,
with `git annex get <file>`, then unlock it, re-editing again or
copying the the file in `.git/annex/bad/` on the unlocked file, then
adding and committing.


## `git-annex` for anonymous users

    git clone http://storage-server.mydomain.com/labdata
	cd labdata
	git annex get <files>
	[...]
	git pull
	git annex get <files>



## Acknowledgment

Thanks to [Michael Hanke's
post](https://github.com/datalad/datalad/issues/335) for inspiring
parts of this tutorial and showing interesting solutions.
