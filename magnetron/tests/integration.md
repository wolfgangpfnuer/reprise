# Magnetron Main Integration Test

This test will go over all of the available features of the command line API
and run them on the local host. To prepare for this, the user executing this
test must have write access to the `/srv/` directory.

The API of the repository class (`magnetron.repository.Repository`) is quite
similar to the command line api implemented in (`magnetron.__main__`), which
is tested separately.

## Initializing the host

Magnetron stores all of the data in `/srv/magnetron`, which we will remove
first of all:

    >>> shutil.rmtree(base_path, ignore_errors=True)

Right now, we can'r create repositories because the basic file structure
hasn't been initialized yet:

    >>> Repository.create("test")
    Traceback (most recent call last):
      ...
    magnetron.repository.RepositoryError: not initialized

We can now use `magnetron.repository.initialize()` to initialize the files
needed on the server:

    >>> os.path.exists(base_path)
    False
    >>> initialize()
    >>> os.path.exists(base_path)
    True

## Managing repositories

After initializing the file system structure, we can start adding repositories:

    >>> repository = Repository.create("test")
    >>> repositories()  # n.b: this is defined in magnetron.repository
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
    ...

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