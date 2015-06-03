---
layout: post
title: "Sync folders with Python"
description: "Manually comparing and synchronizing two folders can be tedious. Add long, confusing and very similar filenames and it's no fun at all. Here's how to automate the process with Python."
tags: [python, script, automation, folder-sync]
---

Manually comparing and synchronizing two folders can be tedious. Add long, confusing and very similar filenames and it's no fun at all.

<figure class="center-align">
	<img src="http://throb.pagesperso-orange.fr/prg/Xojo/SyncTwoFolders_1024.png" title="sync-folders" width="300" height="300" />
	<figcaption>Source: throb.pagesperso-orange.fr</figcaption>
</figure>


We recently faced a similar situation at work. Besides cryptic names, there was also a fair share of twisted logic governing the sync scenarios. We had to get our hands dirty, since the standard tools were useless.

Because we don't have to do the sync often, and the folders have always been of reasonable size, automating the process seemed an overkill. However, as our products grow, so do the folders needing sync. We recently figured spending some time to create a script would be a good future investment.

This post builds upon ideas I came across writing the script, but were ultimately left out or done differently. It concludes with a simple tool capable of syncing two folders. The code was written using version 2.7.6.

### Backing and reverting

Let’s assume nothing is under version control. It would be cool if our script would have a revert mechanism of sorts, in case something funky happens during sync. To keep it simple, maybe a directory snapshot is enough:
 
{% highlight python %}
def backup(directory, tag=None):
    ensure_dir_exists(directory)
    if not os.path.isdir(REPO):
        os.makedirs(REPO)
    assert os.path.isdir(REPO)
    key = str(time.time())
    sha = hashlib.sha1(key).hexdigest()
    archive_path = os.path.join(REPO, sha)
    assert not os.path.isfile(archive_path)
    shutil.make_archive(archive_path, "zip", directory)
    backup_path = os.path.join(directory, BACKUP)
    with open(backup_path, 'w') as b:
        b.write(sha)
    assert os.path.getsize(backup_path) == BACKUP_FILE_LENGTH
    if tag:
        set_tag(directory, archive_path + ".zip", tag)
{% endhighlight %}

It seems dense, but it's easy to explain. First, we create a repository folder or `REPO`, if it doesn’t exist yet. It is where the backup snapshots live. We then save the archived directory under a unique name to `REPO`. Tag is optional, but if given, we'll be able to reference the archived file with it. The name is written into `backup_path` and saved to `directory`. Let's call it the `BACKUP` file. It's there to help us revert:

{% highlight python %}
def revert(directory, archive_path=None):
    ensure_dir_exists(directory)
    if archive_path is None:
        archive_name = get_archive_name(directory)
        archive_path = os.path.join(REPO, archive_name)
    if not os.path.isfile(archive_path):
        raise CpException("Archive '%s' missing." % archive_path)
    shutil.rmtree(directory)
    with zipfile.ZipFile(archive_path) as zf:
        zf.extractall(directory)
{% endhighlight %}

In order to revert, we need to know the archived file to revert to. This is written in the `BACKUP` file which should be part of `directory`. 

`get_archive_name` helps us extract it:

{% highlight python %}
def get_archive_name(directory):
    assert os.path.isdir(directory)
    backup_path = os.path.join(directory, BACKUP)
    if not os.path.isfile(backup_path):
        raise CpException("Backup file missing in '%s'." % directory)
    with open(backup_path) as b:
        archive_name = b.readline()
    if not re.compile(ARCHIVE_NAME_REGEX).match(archive_name):
        raise CpException("Backup file in '%s' corrupted." % directory)
    return archive_name + ".zip"
{% endhighlight %}

Note that any backed state is reachable by a sequence of reverts, as long as `REPO` contains the appropriate archive:

<figure class="center-align">
	<img src="http://i.imgur.com/mIISsct.png" title="backup-revert" />
	<figcaption>Magenta arrows represent backups, green arrows reverts.</figcaption>
</figure>

We should also be able to revert using tags. Here's one way to do it:

{% highlight python %}
INSERT_TAG = "INSERT OR REPLACE INTO tags VALUES (?,?,?)"
GET_TAG = "SELECT dir,zip FROM tags WHERE tag=?"
CREATE_TAGS_TABLE = """
CREATE TABLE IF NOT EXISTS tags (
    tag TEXT PRIMARY KEY,
    dir TEXT,
    zip TEXT
)
"""


def set_tag(directory, archive_path, tag):
    assert tag and os.path.isdir(directory) and os.path.isfile(archive_path)
    with sqlite3.connect(TAGS) as db:
        cursor = db.cursor()
        cursor.execute(CREATE_TAGS_TABLE)
        cursor.execute(INSERT_TAG, (tag, directory, archive_path))
        db.commit()


def revert_by_tag(tag):
    assert tag
    with sqlite3.connect(TAGS) as db:
        cursor = db.cursor()
        cursor.execute(GET_TAG, (tag,))
        result = cursor.fetchone()
    if not result:
        raise CpException("Tag '%s' does not exist." % tag)
    directory, archive_path = result
    revert(directory, archive_path)
{% endhighlight %}

My initial thought was to serialize and de-serialize a dictionary, but performance would degrade quickly. Even with a bit of SQL, I'd argue the above is quite concise.  

It's also quite easy to show tag history of a directory:

{% highlight python %}
GET_DIR = "SELECT tag,zip FROM tags WHERE dir=?"


def show_tag_history(directory):
    with sqlite3.connect(TAGS) as db:
        cursor = db.cursor()
        cursor.execute(GET_DIR, (directory,))
        result = cursor.fetchall()
    if not result:
        raise CpException("Directory '%s' has no backup history." % directory)
    for item in result:
        tag, archive_path = item
        if os.path.isfile(archive_path):
            created = os.path.getctime(archive_path)
            print "TAG=%s, CREATED=%s" % (tag, time.ctime(created))
{% endhighlight %}

There are a few things to consider when reverting:

- we're blindly extracting archives without prior inspection. It is possible that files are created outside of path, e.g. filenames starting with two dots. This could be a security hazard.
- someone could backup the root directory, then try to recover it and happily wipe out the hard drive, since we don't worry about the directory we're clearing.
- if the directory contains symbolic links, `shutil.rmtree(directory)` will throw an `OSError`.

The issues are somewhat easily fixable and might be a good exercise to try out.

### Finding and applying the differences

Finding the differences between two directories couldn't be simpler:

{% highlight python %}
def find_diff(src, dst):
    if os.path.isdir(dst):
        diff = filecmp.dircmp(src, dst)
        diff_list = diff.left_only + diff.diff_files
    else:
        diff_list = [f for f in os.listdir(src)]
    ignore = (BACKUP, SYNC)
    return filter(lambda l: l not in ignore, diff_list)
{% endhighlight %}

If `dst` does not exist, then the difference is the `src` directory content. Otherwise, there's a handy module we can use: `filecmp`. It contains a function `dircmp` that does exactly what we need - it finds all the differences between two folders. 

We're interested in files or folders only in `src`, or common files that differ. We also don't want to copy any of the config files, so we filter them out.

This is how to apply the differences:

{% highlight python %}
def apply_diff(src, dst, diff_list=None, auto_backup=True, backup_tag=None):
    if diff_list is None:
        diff_list = find_diff(src, dst)
    if diff_list:
        if auto_backup:
            backup(dst, backup_tag)
        for item in diff_list:
            src_item = os.path.join(src, item)
            dst_item = os.path.join(dst, item)
            if os.path.isdir(src_item):
                shutil.copytree(src_item, dst_item)
            else:
                shutil.copy(src_item, dst_item)
{% endhighlight %}

The code speaks for itself. The point to note is the backup we perform before any copying is done. This enables us to revert if something goes sour.

### Syncing the same folders over and over and over...

Sometimes, you know beforehand the folders you need to sync. For example, you know that folder `A` will always have to be synced with folders `B` and  `C`. This is where `SYNC` file comes into play. It contains one or more source folders, each listed on a separate line. 

In the example above, folder `A` should contain the `SYNC` file with the following content:

{% highlight bash %}
/an/absolute/path/to/B
/an/absolute/path/to/C
{% endhighlight %}

Then, all we need to do is sync the directory containing the `SYNC` file:

{% highlight python %}
def sync(directory, backup_tag=None):
    ensure_dir_exists(directory)
    sync_path = os.path.join(directory, SYNC)
    if not os.path.isfile(sync_path):
        raise CpException("Sync file for '%s' missing." % directory)
    with open(sync_path) as s:
        src_list = s.read().splitlines()
    if not src_list:
        raise CpException("Sync file is empty.")
    for src in src_list:
        ensure_dir_exists(src, "Invalid source dir: '%s'." % src)
    backup(directory, backup_tag)
    for src in src_list:
        apply_diff(src, directory, auto_backup=False, backup_tag=backup_tag)
{% endhighlight %}

As you can see, it's as straightforward as opening the `SYNC` file, reading the sources, and then applying the differences. 

Of course, we should provide means to generate such file:

{% highlight python %}
def generate_sync_file(directory, sources):
    ensure_dir_exists(directory)
    absolute_paths = [os.path.abspath(s) for s in sources]
    sync_file = os.path.join(directory, SYNC)
    with open(sync_file, "w") as cp:
        cp.writelines(absolute_paths)
    assert os.path.getsize(sync_file) > 0
{% endhighlight %}

### Adding CLI

To wrap what we've done in a simple utility tool, we should create a command line interface, so the user can interact with it. `argparse` module makes this simple. Before turning to code, here's what the user should be able to do:

#### 1. Copy different files from one directory to another

Example usage: `cp -t sample_tag /source/path/dir /destination/path/dir`

The comand requires a `src` directory and a `dst` directory, where `dst` will be synced with `src`. Tag is optional.

#### 2. Revert a directory or tag 

Example usage: `rv -t sample_tag` or `rv -d /random/path/dir`

#### 3. Sync a directory

Example usage: `sync -t just_in_case /random/dir/path`

`/random/dir/path` should contain the `SYNC` file. Tag is optional.

#### 4. Create a `SYNC` file

Example usage: `mksync dir/path/where/sync/is/created /fst/src /snd/src /trd/src`

Create a `SYNC` file in the first specified directory. Any directory listed afterwards is added to the source list.

#### 5. Show tag history

Example usage: `th /random/dir/path`

It shows the available revert tags.

Here's the above in code:

{% highlight python %}
def diff_parser():
    parser = argparse.ArgumentParser(
        description="Compare and copy missing files from one dir to another"
    )

    subparsers = parser.add_subparsers(title="Options", dest="opt")
    tag_parser = argparse.ArgumentParser(add_help=False)
    tag_parser.add_argument("-t", "--tag", help="Backup tag")

    cp = subparsers.add_parser("cp", help="Copy from one dir to another",
                               parents=[tag_parser])
    cp.add_argument("src", help="source dir", action=ValidDirAction)
    cp.add_argument("dst", help="destination dir", action=ValidDirAction)

    rv = subparsers.add_parser("rv", help="Revert")
    rv.add_argument("-d", "--dir", help="dir to revert", action=ValidDirAction)
    rv.add_argument("-t", "--tag", help="tag to revert")

    sync = subparsers.add_parser("sync", help="Auto-sync", parents=[tag_parser])
    sync.add_argument("dir", help="dir to sync", action=ValidDirAction)

    mksync = subparsers.add_parser("mksync", help="Generate auto-sync folder")
    mksync.add_argument("dir", help="dir to auto-sync", action=ValidDirAction)
    mksync.add_argument("src", nargs='+', help="source dirs",
                        action=ValidDirAction)

    history = subparsers.add_parser("th", help="Tag history")
    history.add_argument("dir", help="dir to check", action=ValidDirAction)

    return parser
{% endhighlight %}

Perhaps the only interesting thing in our parser is the custom action:

{% highlight python %}
class ValidDirAction(argparse.Action):
    def __call__(self, parser, namespace, values, option_string=None):
        directories = values if isinstance(values, list) else [values]
        for d in directories:
            if not (os.path.isdir(d) and os.access(d, os.W_OK)):
                raise argparse.ArgumentError(self, "Invalid dir: %s" % d)
        setattr(namespace, self.dest, values)
{% endhighlight %}

It ensures that any path we provide as an argument is an existing directory we can write to. 

### Fin

We can now sync two folders and revert if need be. The tool is very simple and only offers crude functionality, but it's a good starting point to build upon. What always amazes me is the expressiveness of Python and what can be achieved with
cca. 200 lines of code, half of which are paranoid asserts and param checks. 

I'd also like to note that although I love Python, I don't use it enough to consider myself a _pythonista_. If you spot any piece of code that can be replaced with a more standarized idiom, please let me know!

As always, there's a GitHub repo where you can find the complete script.
