This is a library for creating and inspecting
[HFS](https://en.wikipedia.org/wiki/Hierarchical_File_System)-format
disk images. Mac-specific concepts like [resource
forks](https://en.wikipedia.org/wiki/Resource_fork) and
[type](https://en.wikipedia.org/wiki/Type_code)/[creator
codes](https://en.wikipedia.org/wiki/Creator_code) are first-class
citizens.

Python interface
----------------
The Python API is simple. The contents of a `Volume` or a `Folder` are
accessed using the index operator `[]`. While working on a filesystem,
its entire high-level contents are stored in memory as a Python object.

```
from machfs import Volume, Folder, File

v = Volume()

v['Folder'] = Folder()

v['Folder']['File'] = File()
v['Folder']['File'].data = b'Hello from Python!\r'
v['Folder']['File'].rsrc = b'' # Use the macresources library to work with resource forks
v['Folder']['File'].type = b'TEXT'
v['Folder']['File'].creator = b'TTXT' # Teach Text/SimpleText

with open('FloppyImage.dsk', 'wb') as f:
    flat = v.write(
        size=1440*1024, # "High Density" floppy
        align=512, # Allocation block alignment modulus (2048 for CDs)
        desktopdb=True, # Create a dummy Desktop Database to prevent a rebuild on boot
        bootable=True, # This requires a folder with a ZSYS and a FNDR file
        startapp=('Folder','File'), # Path (as tuple) to an app to open at boot
    )
    f.write(flat)

with open('FloppyImage.dsk', 'rb') as f:
    flat = f.read()
    v = Volume()
    v.read(flat) # And you can read an image back!
```

Command-line interface
----------------------
This package also installs the `MakeHFS` and `DumpHFS` utilities, for
working with folders on your native filesystem. Briefly, resource forks
are stored in Rez-formatted `.rdump` files, and type and creator codes
are stored in 8-byte `.idump` files. Admittedly this method of storage
is not pretty, but it exposes changes to resource files without
requiring Mac-specific software. For example, Git can track the addition
and removal of resources. Files with a `TEXT` type are assumed to be
UTF-8 encoded with Unix-style (LF) line endings, and are converted to
Mac OS Roman encoding with Mac-style (CR) line endings.

Both commands have a `--help` argument to display their options.

Why?
----
I want an automated, reproducible way to compile legacy MacOS software.
Without any current operating system fully supporting HFS,
[libhfs/hfsutils](https://www.mars.org/home/rob/proj/hfs/) (a C library
and command-line wrapper) is the most capable implementation. The
implementor chose to emulate POSIX I/O on a fake "mounted" filesystem.
While this is important for machines with very limited RAM, the
maintenance of consistent HFS data structures across incremental
operations is a complicated task requiring a large amount of low-level
code. Frequent I/O to the real filesystem also occurs. Current machines
have memory and cycles to burn, so an in-memory implementation in a
high-level programming language seemed like a reasonable tradeoff. As a
result, `machfs` has nearly an order of magnitude fewer lines than
`libhfs`, and is more maintainable, at a nearly negligible cost in
performance.
