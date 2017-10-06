zzzfoo: Desktop file search with Rofi and Recoll
================================================

This script lets you combine the excellent full-text search tool [Recoll](https://www.lesbonscomptes.com/recoll/)
with [Rofi](https://github.com/DaveDavenport/rofi) (popular dmenu replacement,
among other things) to quickly search all your indexed files. It simply
performs a Recoll search using  Recoll's Python module, pipes the output to
`rofi -dmenu`, and (optionally) does something with the selected match.

If you only need file paths in your results, forget this script and just grep/sed
the result of `recoll -t -e -b <foobar>` and pipe that into rofi or dmenu; this
script is basically a one-liner that got out of hand. However, if you want
titles and MIME types and highlighted extracts and colors, as well as
various options, keep reading.

WARNING: Still experimental, use at your own risk, etc.

![zzzfoo screenshot](https://anders.unix.se/images/zzzfoo_01.png)

![zzzfoo screenshot](https://anders.unix.se/images/zzzfoo_02.png)

Requirements
------------

  * Python 2 or 3
  * [Recoll](https://www.lesbonscomptes.com/recoll/) (and its Python module), tested with 1.21/1.23
  * [Rofi](https://github.com/DaveDavenport/rofi) 1.3+

Many Linux distributions (as well as FreeBSD) package both. For e.g. Debian/Ubuntu:

```sh
apt install recoll python-recoll rofi # or python3-recoll
```

Note that Ubuntu 16.04 has Rofi 0.15.x which is very outdated. You need 1.3 or later.
See Rofi's [installation guide](https://github.com/DaveDavenport/rofi/blob/next/INSTALL.md).

Start `recoll` and let it index. (Recoll full-text indexes a bunch of file
types natively; others need [various helper programs](http://www.lesbonscomptes.com/recoll/features.html#doctypes),
which may or may not be installed.)

Installation
------------

Just drop `zzzfoo` wherever you like (somewhere in your
`$PATH`, perhaps) and make it executable (`chmod +x zzzfoo`).

Usage
-----

If run without arguments it will launch a Rofi search dialog, search using the
query you enter, present the results, and write the absolute path of your
chosen match to STDOUT. That's of limited use, but there are bunch of options --
example:

Open selected match with `xdg-open`:

    zzzfoo -o xdg-open

_(Do NOT -o something_that_deletes_files; there are certainly bugs. See
[the Arch wiki](https://wiki.archlinux.org/index.php/default_applications#xdg-utils)
for a reminder on how to set default applications for different MIME types.)_

Exclude files of type message/rfc822 (emails), strip `/home/foobar/` from the
_displayed_ file path in results list, open selected file with `xdg-open`:

    zzzfoo -e ' -mime:message/rfc822' -p /home/foobar/ -o xdg-open

_(See Recoll's manual section about [the query language](http://www.lesbonscomptes.com/recoll/usermanual/usermanual.html#RCL.SEARCH.LANG).)_

Pass some options to Rofi -- here window at top right, 50% width:

    zzzfoo -r '-location 3 -width 50'`



Run query directly (and show results in Rofi) without using the search dialog,
light grey color for abstracts:

    zzzfoo -q foobar --color-abstract '#777'

    zzzfoo -q 'quote if multiple search terms' --color-abstract '#777'

Show synthetic abstracts (i.e. constructed by extracting text around search terms;
the indexed abstract, shown by default, is often just the beginning of the file):

    zzzfoo --synthetic-abstract

_(Note: this is a bit slower.)_

Run the script with `-h` to see all options.

See http://www.lesbonscomptes.com/recoll/usermanual/usermanual.html#RCL.SEARCH.LANG for
information about the query language.

For theming Rofi, see https://github.com/DaveDavenport/rofi-themes.

I suggest binding zzzfoo to something convenient. For example, for i3 I have the
following in `~/.i3/config`:

    bindsym $mod+space exec --no-startup-id ~/bin/zzzfoo -p /home/andersju/ -e ' -mime:message/rfc822' -o xdg-open

Messy Python-free sed-n-awk alternative
---------------------------------------

Just for fun (hmm), or if you really don't want to use Python, here's a
script that simply parses the output of Recoll's text mode and provides
something similar to the Python script. Note that it will break if Recoll
changes its default output format, or perhaps due to certain characters in
file names, or for other reasons. Use the Python version.

(It's possible to use the fact that Recoll can output base64-encoded fields
which would be much easier to handle, but that's a bit slower. Some of the
weirdness below is required for it to work with both GNU and BSD versions of
tools, and with plain bourne shells.)

```sh
#!/bin/sh

PATH_TO_STRIP="/home/foobar/"
rofi_search=$(rofi -dmenu -p 'Search: ')

if [ -n "${rofi_search}" ]; then
  file_path=$(recoll -t -A -e -n 20 "${rofi_search}" \
    | tail -n +3 \
    | sed 'N;N;N;s/\n/ /g' \
    | sed 's/&/\&amp;/g; s/</\&lt;/g; s/>/\&gt;/g; s/"/\&quot;/g' \
    | sed 's/'"'"'/\&#39;/g; s/&ldquo;/\"/g; s/&rdquo;/\"/g' \
    | awk 'BEGIN {FS="\t"; OFS=FS}
           {{gsub(/^ ABSTRACT  |\/ABSTRACT/, "", $6)}}
           {print "<b>" substr($3,2,length($3)-2) "</b> <i>" $1 "</i>";
            print "ZZZBAR" $2; print $6 "\n\n" $2 "ZZZFOO"}' \
    | tr -d '\t' \
    | awk '{gsub(/ZZZFOO$/,"\t");}1' \
    | sed "s#^ZZZBAR\[file://${PATH_TO_STRIP}\(.*\)\]#\1#" \
    | awk '{gsub(/^\[file:\/\//,"");}1' \
    | awk '{gsub(/\]\t/,"\t");}1' \
    | sed 's/+/ /g;s/%/\\x/g' \
    | xargs -0 printf "%b" \
    | sed -e ':a' -e 'N' -e '$!ba' -e 's/	\n/	/g' \
    | sed '$ s/.$//' \
    | rofi -dmenu -eh 3 -sep '\t' -markup-rows -p "Filter results: " \
    | tail -n 1 \
    | sed 's/&amp;/\&/g; s/&lt;/</g; s/&gt;/>/g; s/&quot;/"/g; s/&#39;/'"'"'/g' \
    | tr -d '\n')

  if [ -n "${file_path}" ]; then
    xdg-open "${file_path}"
    exit 0
  fi
fi
```

(Note that the third-to-last sed with `'s/	\n/	/g'` contains two actual tabs,
not spaces. Be careful so that your editor doesn't replace these with spaces
when pasting. `\t` would've been more convenient but alas, can't use that
(at least not in the replacement) because POSIX. I think.)
