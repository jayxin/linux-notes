# Package Manager

## Red Hat Series

### How to extract files from a RPM package?

```sh
mkdir packagecontents; cd packagecontents

# -i extract
# -d make directories
# -m preserve modification time
# -v verbose
rpm2cpio ../foo.rpm | cpio -idmv

find .
```
