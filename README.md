## Description

Below are instructions about how to download locally all (hashed) passwords that are known to be leaked into the https://haveibeenpwned.com so you can check if a password is leaked without using their remote API.

**Note:** There is no security issue using the HIBP remote API, as you are not sending the password to HIBP, but instead you just "ask" to the API for the SHA1 passwords starting with 5 Hex digit of the SHA1 of a password.

Example: if your password is `lauragpe`, all that HIBP would know about you, is that are asking for SHA1 hashes that start with `21BD1`. Your password may be anything including `melobie` which also start with `21BD1`



API information is available on https://haveibeenpwned.com/API/v3#SearchingPwnedPasswordsByRange

> In order to protect the value of the source password being searched for, Pwned Passwords also implements a [k-Anonymity model](https://en.wikipedia.org/wiki/K-anonymity) that allows a password to be searched for by partial hash. This allows the first 5 characters of a SHA-1 password hash (not case-sensitive) to be passed to the API (testable [by clicking here](https://api.pwnedpasswords.com/range/21BD1)):
>
> ```
> GET https://api.pwnedpasswords.com/range/{first 5 hash chars}
> ```
>
> When a password hash with the same first 5 characters is found in the Pwned Passwords repository, the API will respond with an HTTP 200 and include the *suffix* of every hash beginning with the specified prefix, followed by a count of how many times it appears in the data set. The API consumer can then search the results of the response for the presence of their source hash and if not found, the password does not exist in the data set. A sample response for the hash prefix "21BD1" would be as follows:
>
> ```
> 0018A45C4D1DEF81644B54AB7F969B88D65:1 00D4F6E8FA6EECAD2A3AA415EEC418D38EC:2 011053FD0102E94D6AE2F8B83D76FAF94F6:1 012A7CA357541F0AC487871FEEC1891C49C:2 0136E006E24E7D152139815FB0FC6A50B15:2 ...
> ```
>
> A range search typically returns approximately 500 hash suffixes, although this number will differ depending on the hash prefix being searched for and will increase as more passwords are added. 
>
> There are 1,048,576 different hash prefixes between 00000 and FFFFF (16^5) and every single one will return HTTP 200; there is no circumstance in which the API should return HTTP 404.





## Why not using the HIBP "official" tool to download locally

HIBP provide for now this repo:  https://github.com/HaveIBeenPwned/PwnedPasswordsDownloader

> It's just 16^5 separate requests and the responses could be dumped into a big text file, how hard could it be?

As per informations found on https://www.troyhunt.com/downloading-pwned-passwords-hashes-with-the-hibp-downloader/

But that above "official" version seems to require some "DotNet", some non free software OS,  etc.. and contains too many lines of code for me to read and understand.



This other version just require `crunch` and `aria2` as "additional" packages, and basically contains only few  lines of "code" easy to read/understand/customize.

So there is not even "code" to be downloaded from this repo.. which is only a `README` about how to proceed by yourself.



## Information

In short this is what we are doing here

- Create a list of all URLs that contains sha1 hashes (using `crunch` and `sed`)

This step takes only few seconds and are needed only once. (Could even be included here, but why not "build" it yourself)

- Download all those 1 048 576 files with the HTTP client `aria2`
- Once each small file is downloaded, it apply small optional modifications
  - Add CR at the end of the file
  - Add the 5 digit prefix into the file itself
  - Compress each file

This step on my server takes around 30 / 60 minutes.



Downloading a local copy instead of requesting the remote API may be helpful to:

- be used in restricted environments without a permanent access to Internet
- not be dependent of the network or of the availability access of the remote API
- etc...

## Prerequisites

- Require 21 GB of free space to store downloaded files

```
$ du -s .
20984244
```



- Generate all the 5 Hex digit combinations

​          16^5  =  1048576

with `crunch` (or any other tool/language you like and can do it as fast as crunch)

> crunch - generate wordlists from a character set

```
$ sudo apt-get install crunch
$ crunch 5 5  0123456789ABCDEF > out
Crunch will now generate the following amount of data: 6291456 bytes
6 MB
0 GB
0 TB
0 PB
Crunch will now generate the following number of lines: 1048576 
$ wc -l out
1048576 out
$ shuf -n 5 out
8584C
77908
1FEFF
55E89
0DA1C
```

- Add the URL in front

```
$ sed 's|^|https://api.pwnedpasswords.com/range/|g' out > urls.txt
$ shuf -n 5 urls.txt
https://api.pwnedpasswords.com/range/C32A3
https://api.pwnedpasswords.com/range/E8C0E
https://api.pwnedpasswords.com/range/30F83
https://api.pwnedpasswords.com/range/D5209
https://api.pwnedpasswords.com/range/ED6CF
```



- Install `aria2` 

  > aria2c - The ultra fast download utility

```
sudo apt-get install aria2
```



- Create `hook.sh` that will be executed once file is downloaded

```
#!/bin/bash

# aria2 passes 3 arguments to specified command when it is executed. These arguments are: GID, the number of files and file path. 

# echo "Called with [$1] [$2] [$3]"
# aria2c --on-download-complete hook.sh http://example.org/file.iso
# Called with [1] [1] [/path/to/file.iso]

# Add CR
echo "" >> $3

FILENAME=$(basename $3)
# add 5 digit prefix into the file itself
sed -i "s/^/${FILENAME}/g" $3

# gzip in case it help
gzip $3
```





## Usage

- Adjust parameters and start download (possibly better inside a `screen`)

```
$ time aria2c --input-file=/data/HIBP/urls.txt --max-concurrent-downloads=128 --connect-timeout=60 --max-connection-per-server=16 --split=2 --min-split-size=1M --human-readable=true --download-result=full --file-allocation=none --deferred-input --on-download-complete /data/HIBP/hook.sh --dir=/mnt/disk/HIBP
```

It takes around 30 / 60 minutes on my server.



- Once completed, in this example all files are stored into `/mnt/disk/HIBP/` directory

```
$ ls | head
00000.gz
00001.gz
00002.gz
00003.gz
00004.gz
00005.gz
00006.gz
00007.gz
00008.gz
00009.gz
```

```
$ du -hs .
21G
```





- Check for a password into the local "database"

```
$ echo -n "password123" | sha1sum |  tr [:lower:] [:upper:]
CBFDAC6008F9CAB4083784CBD1874F76618D2A97  -
$ zgrep CBFDAC6008F9CAB4083784CBD1874F76618D2A97 /mnt/disk/HIBP/CBFDA.gz
CBFDAC6008F9CAB4083784CBD1874F76618D2A97:248071
```

The password  `password123` has leaked 248071 times



- Or use this kind of wrapper (to possibly be improved)

```
#!/bin/bash

PASSWORD="${1}"
DIR="/mnt/disk/HIBP"

SHA1=$(echo -n "${PASSWORD}" | sha1sum | tr [:lower:] [:upper:] | awk '{print $1}')
FILENAME=$(echo -n "${PASSWORD}" | sha1sum |  tr [:lower:] [:upper:] | awk '{print $1}' | fold -5 | head -1)
GZ_FILENAME=${DIR}/${FILENAME}.gz

TMP=$(mktemp)
zgrep "${SHA1}" ${GZ_FILENAME} > ${TMP}

if [[ -s ${TMP} ]]
then
        echo -n "🔥 This password leaked 🔥  "
        cat ${TMP}
else
        echo "✅ This password looks not leaked yet ✅"
fi

rm ${TMP}
```

```
$ ./check.sh "Tr0ub4dor&3"
🔥 This password leaked 🔥  874572E7A5AE6A49466A6AC578B98ADBA78C6AA6:4

$ ./check.sh "Szdle87d-sfhk232é"
✅ This password looks not leaked yet ✅

$ ./check.sh "correcthorsebatterystaple"
🔥 This password leaked 🔥  BFD3617727EAB0E800E62A776C76381DEFBC4145:216
```



Same result as requesting the remote API over the network 👌:

```
$ curl -s https://api.pwnedpasswords.com/range/BFD36 | grep 17727EAB0E800E62A776C76381DEFBC4145
17727EAB0E800E62A776C76381DEFBC4145:216
```

