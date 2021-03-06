# Reprise Main Integration Test

This test will go over all of the available features of the command line API
and run them on the local host. To prepare for this, the user executing this
test must have write access to the `/srv/` directory.

The API of the repository class (`reprise.repository.Repository`) is quite
similar to the command line api implemented in (`reprise.__main__`), which
is tested separately.

## GPG

Before we get into Reprise itself, we need to look at GPG. Reprise uses
the default GPG key pair to sign packages, and any user wanting to download
packages needs to add the public key using `apt-key`.

    >>> print(get_default_public_key())
    -----BEGIN PGP PUBLIC KEY BLOCK-----
    Version: GnuPG v1...
    <BLANKLINE>
    mQINBFGTQwgBEADfsu4HiJwgsCJggMmK+WFnppltqE...

    >>> get_default_key_id()
    '3F479202'

## Initializing the host

Reprise stores all of the data in `/srv/reprise`, which we will remove
first of all:

    >>> shutil.rmtree(base_path, ignore_errors=True)

Right now, we can'r create repositories because the basic file structure
hasn't been initialized yet:

    >>> Repository("test")
    Traceback (most recent call last):
      ...
    reprise.repository.RepositoryError: not initialized
    >>> Repository.create("test")
    Traceback (most recent call last):
      ...
    reprise.repository.RepositoryError: not initialized

We can now use `reprise.repository.initialize()` to initialize the files
needed on the server:

    >>> os.path.exists(base_path)
    False
    >>> initialize()
    >>> os.path.exists(base_path)
    True
    >>> initialize()
    Traceback (most recent call last):
      ...
    reprise.repository.RepositoryError: already initialized

## Managing repositories

After initializing the file system structure, we can start adding repositories:

    >>> repository = Repository.create("test")
    >>> repositories()  # n.b: this is defined in reprise.repository
    [<Repository 'test'>]


A repository also comes with its configuration, generated from templates:

    >>> os.listdir(repository.path)
    ['conf']
    >>> with open(os.path.join(repository.path, "conf", "distributions")) as f:
    ...     print(f.read())
    ...
    Origin: ...
    Label: test
    ...
    SignWith: ...

    >>> Repository("does-not-exist")
    Traceback (most recent call last):
      ...
    reprise.repository.RepositoryError: repository doesn't exist

We can delete repositories by using their `expunge()` method:

    >>> repository.expunge()
    >>> os.path.exists(repository.path)
    False
    >>> repository.expunge()
    Traceback (most recent call last):
      ...
    FileNotFoundError: ...
    >>> repositories()
    []

Let's create a repository and add a package:

    >>> Repository("../this-name-is-invalid")
    Traceback (most recent call last):
      ...
    reprise.repository.RepositoryError: invalid name ...

    >>> Repository("incoming")
    Traceback (most recent call last):
      ...
    reprise.repository.RepositoryError: invalid name ...

    >>> filename = glob.glob("/var/cache/apt/archives/pep8_*_all.deb")[0]
    >>> repository = Repository.create("pep8")

    >>> repository.add("?")
    Traceback (most recent call last):
      ...
    FileNotFoundError: [Errno 2] ...

    >>> repository.add(filename)

    >>> list(repository.packages())
    [<Package '...|main|amd64: pep8 ...>, ...]

    >>> list(repository.get("pep8"))
    [<Package '...|main|amd64: ...'>, ...]

    >>> repository.get("does-not-exist")
    Traceback (most recent call last):
      ...
    reprise.repository.RepositoryError: package not found

    >>> Repository.create("pep8")
    Traceback (most recent call last):
      ...
    reprise.repository.RepositoryError: repository exists

Of course we can remove packages:

    >>> repository.remove("pep8")
    >>> list(repository.packages())
    []
    >>> repository.add(filename)

When we have a repository, we can use "pull" to update one repository to
the state of another one. This works by overwriting the entirety of the
contents of the target repository with those of the source repository.
This process is used, for example, to update a "stable" repository using
the "unstable" repository as a source.

    >>> unstable = repository
    >>> stable = Repository.create("pep8-stable")

    >>> {i.name for i in unstable.packages()}
    {'pep8'}
    >>> {i.name for i in stable.packages()}
    set()

    >>> stable.pull(unstable)

    >>> {i.name for i in unstable.packages()}
    {'pep8'}
    >>> {i.name for i in stable.packages()}
    {'pep8'}

    stable.expunge()

## Pulling remote repositories

It is possible to pull packages from a remote server as far as ssh access to
the remote host is given. Firstly we do a "dry run":

    >>> remote = Remote("vagrant", "localhost", "pep8", "pep8-synced")
    >>> remote.packages()
    receiving incremental file list
    pep8/
    pep8/conf/
    pep8/conf/distributions
    ...
    pep8/dists/
    ...
    pep8/pool/main/p/pep8/pep8...all.deb
    <BLANKLINE>
    sent ... bytes  received ... bytes  ... bytes/sec
    total size is ...  speedup is ... (DRY RUN)

This will synchronize the two repositories (`pep8` and
`rsync://localhost/../pep8-synced`):

    >>> Repository("pep8-synced")
    Traceback (most recent call last):
      ...
    reprise.repository.RepositoryError: repository doesn't exist

    >>> remote.synchronize()
    >>> {i.name for i in Repository("pep8-synced").packages()}
    {'pep8'}
