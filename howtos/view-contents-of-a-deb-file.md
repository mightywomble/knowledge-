# View contents of a Deb file

Its useful to be able to see what is actually packaged in a deb file, these instructions will explain how to download a Deb File and “open” it up from a Debian Command line

### PreReqs

As Root

Create a folder somewhere called debfile

```jsx
mkdir debfile
cd debfile
```

### Install Software

We need the ar command and the ability to un tar an xr file

```jsx
apt install binutils xz-utils
```

### Download the Deb File (into the debfile folder)

```jsx
apt download cudos-noded
```

This will download the package file with the full name into the debfile directory

```jsx
Example: cudos-noded_1.1.2-146.el8_amd64.deb
```

### Decompress the file

```jsx
 ar x cudos-noded_1.1.2-146.el8_amd64.deb
```

This will create 3 files

```jsx
-rw-r--r-- 1 root root 1304 Apr 17 16:38 control.tar.xz
-rw-r--r-- 1 root root 8172 Apr 17 16:38 data.tar.xz
-rw-r--r-- 1 root root    4 Apr 17 16:38 debian-binary
```

#### control.tar.xz

To decompress this file run

```jsx
tar -xf control.tar.xz
```

This will display the files which tell apt what to do with the files in data.tar.xz

```jsx
**-rw-r--r-- 1 root root   85 Jan 11 17:49 conffiles
-rw-r--r-- 1 root root  343 Jan 11 17:49 control
-rw-r--r-- 1 root root  441 Jan 11 17:49 md5sums
-rwxr-xr-x 1 root root 1298 Jan 11 17:49 postinst
-rwxr-xr-x 1 root root  312 Jan 11 17:49 preinst**
```

For example control contains the meta data you’ll see when doing an apt investigation on the deb file

```jsx
Package: cudos-noded
Version: 1.1.2-146.el8
Architecture: amd64
Maintainer: Jenkins User <jenkins@private-testnet-full-node-01>
Installed-Size: 60
Depends: cosmovisor
Section: alien
Priority: extra
Description: Cosmovisor Node Client Files - cudos
 Cosmovisor client files for - cudos
 .
 (Converted from a rpm package by alien version 8.95.)
```

#### data.tar.xz

To decompress this file run

```jsx
tar -xf data.tar.xz
```

this will display all the files in the data structure they should be placed in the linux file system

```jsx
drwxr-xr-x 4 root root 4096 Jan 11 17:48 etc
drwxr-xr-x 4 root root 4096 Jan 11 17:48 usr
drwxr-xr-x 3 root root 4096 Jan 11 17:48 var
```

#### debian-binary

Sets a Marker for the version of Devian Binary to use to install the file

```jsx
cat debian-binary 
2.0
```
