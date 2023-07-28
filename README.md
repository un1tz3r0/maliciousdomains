# maliciousdomains
Quickly collect, combine and clean a masterlist of malicious domains.

I present some tiny shell snippets that can be used as a rudimentary malware detector. Why? Because supply-chain attacks are a very real problem. Given the fragmented landscape of the open-source package manager world, I am regularly pulling and running code which I *assume* is safe from at least 4 or 5 different sources, most of which are open to self-published packages. If that thought doesn't scare you a little, I'd say you've got too much faith in humanity. There are bad actors out there, trust me. Almost as many as there are decent, honest people. And they are often not who you'd expect.

But on a more positive note, there's plenty you can do to stay safe(ish). One thing is automate your own audits of the software you're running. A rudimentary check for malicious code might scan installed scripts and binaries for the names of known malicious domains, where they might upload data collected from your computer or download commands and payloads to further compromise your system.

But how do we know which domains to look for? Well, there are plenty of lists. We need a list of them. Preferably direct links to current-versions in text format. There's a great repo which contains just that right [here](https://github.com/PeterDaveHello/threat-hostlist). Many thanks to [@PeterDavidHello](https://github.com/PeterDavidHello) for maintaining this list. It's perfect for our needs, and we can grab a copy and pipe it to stdout like this:

```bash
wget 'https://github.com/PeterDaveHello/threat-hostlist/raw/master/README.md' -O -
```

Now, this is wonderful and nicely readable especially with a markdown processor. Markdown is just delightful in many ways, but in order for our script to do anything with the sites on this list, we need to extract the links with `sed` by piping `wget's` output into it like so:

```bash
... | sed -ne 's/\[.*\.raw\]://p'
```

This gets all of the .raw links out of the markup, which we will want to loop through and download, which can be accomplished by piping that output to `xargs` invoking `wget` on each:

```bash
... | xargs wget --timeout 3 --tries 2 -O -
```

This will try and grab each url piped to it, limiting connection attempts to 3s before trying again, and if the second attempt fails, it will skip that URL and move on to the next, which is helpful to avoid getting stuck waiting for an unresponsive or DDOSed server (this happens with sites hosting these lists often).

The fetched lists are written to stdout, which are in various 
formats but most will be compatible with the bind zonefile format, intended to be used by network administrators who run a forwarding DNS server that 
is used by machines on an intranet. This format will have two columns, an ip-address and then the domain to map to it. The ip address in these blacklist zonefiles is usually just 0.0.0.0, which effectively blocks access to the domain by resolving it to a non-routable address. We want to strip out these bogus IPs and just get the domains, and `sed` does this nicely:

```bash
... | sed -e 's/^[0-9]\+\.[0-9]\+\.[0-9]\+\.[0-9]\+[ \t]//'
```

Now we want to filter out anything else that doesn't look like a domain. Some of these lists will have some boilerplate or comments or a disclaimer, so we can use `grep` with a regexp that maches on word boundaries, outputting only matching text, to find everything that looks like a domain name. To strip off the line numbers in `grep`'s output, we can pipe it through `cut`, like this:

```bash
... | grep -niobe '\b[a-zA-Z0-9\\.-]\+\.[a-zA-Z0-9\\.]\+\b' - | cut -d ':' -f 3-
```

Now we just need to dedupe and remove some common top level domains that are not malicious but are there because they are in some lists that include malicious URLs. To dedupe, we use `sort -u` and then a final inverted match `grep` to whitelist a handful of safe TLDs:

```bash
... | sort -u \
| grep -vwFf whitelist.txt \
> baddomains.txt
```

## Scanning for malware

Now that we have baddomains.txt, we can look for files which refer to one of these domains, which are pretty good candidates for being malware. Granted, it is super easy to obfuscate a string in a script or binary, which would render this technique useless (a better way would be to scan the allocated memory of all loaded programs, or filter outgoing DNS lookups or TCP and UDP connections. I'll leave that for another time tho...). So to scan for files which contain one of these domains, `grep` works quite well. We use its pattern-file option (`-f`) instead of specifying patterns on the command line like usual, to read the search strings from our bad domains list. The `-r` flag specifies to recursively scan all files in directories beneath the given paths, so for instance to scan all python packages installed in our user's home directory, which is where `pip` will place them if you are not running it as root (something you wouldn't dream of ever doing, riiiiiight?), we can have grep search under `~/.local/lib` and `~/.local/bin` like this:

```bash
grep -wFf baddomains.txt -r ~/.local/lib/
```
