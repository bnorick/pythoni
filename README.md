# pythoni

I wrote this tool because I always find myself fighting with bash: struggling to remember a command's syntax to do something simple or needing to resort a search to determine whether bash even has a command to do what I need, all the while knowing that I could do it quickly in Python. So why not do it in Python? Well, it's nice to mix bash and Python via piping, but that quickly becomes a mess from a Python script's perspective, and on and on.

I wanted to solve this problem and boost my own productivity, so I wrote `pythoni` (short for "python, interactive").

### What can it do?
- Pipe from bash into `pythoni`, either executing simple transformations or more complex multi-line code blocks. If you write to stdout, then you can keep piping it on down the line.
- Pipe into `pythoni` with no args for an interactive REPL where you can access the local variable `stdin` which contains list of lines (including trailing `\n` characters) which were piped in. (*In the future, from the REPL you will be able to save a subset of the actions you performed in the interactive session as a new script or as a copy-pastable `pythoni` chunk. It's sorta kinda there now, but it's a bit buggy.*)


### How to use it.
Simple things
```
$ echo -e '1\n2\n3' | pythoni -p '"".join(stdin).rstrip().replace("\n", "-")'
1-2-3
```
Multiple prints
```
$ echo -e '1\n2\n3' | pythoni -p '"".join(stdin).rstrip().replace("\n", "-")' -p 'f"input_lines={len(stdin)}"'
1-2-3
input_lines=3
```

Find yourself using fstrings a lot? Great, use `--print-fstring` (shorthand `-pf`). Note that literal single quotes (') must be escaped if used.
```
$ echo -e '1\n2\n3' | pythoni -pf 'input_lines={len(stdin)}'
input_lines=3

$ echo -e '1\n2\n3' | pythoni -pf "\'input_lines\'={len(stdin)}"  # escaping
'input_lines'=3
```

You can mix `-p` and `-pf` (as well as `-l` and `-c`, discussed later) as necessary
```
$ echo -e '1\n2\n3' | pythoni -p '"".join(stdin).rstrip().replace("\n", "-")' -pf 'input_lines={len(stdin)}' -p '"foobar"'
1-2-3
input_lines=3
foobar
```

Maybe a `lambda` which takes as input each line and can drop lines by returning `None` would be helpful? Use `--lambda` (shorthand `-l`). To be valid, a lambda must take a single argument.
```
$ echo -e '1\n2\n3' | pythoni -l 'lambda line: line if "2" in line else None'
2
```

More complicated tasks might need multi-line code segments via `--code` (shorthand `-c`), there are a few ways to use it.

Option 1 – read code an env var and (potentially re-)use it
```
$ read -r -d '' PYTHONI_CODE <<'EOF'
for line in stdin:
    print(f"{int(line)*100}")
EOF
$ echo -e '1\n2\n3' | pythoni -c "${PYTHONI_CODE}"
100
200
300
$ echo -e '4\n5\n6' | pythoni -c "${PYTHONI_CODE}"
400
500
600
```

Option 2 – send code directly in via process substitution
```
$ echo -e '1\n2\n3' | pythoni -c <(cat <<'EOF'
for line in stdin:
    print(f"{int(line)*100}")
EOF
)
100
200
300
```

### Examples

Process `ripgrep` output to pretty-print words in many languages matching a pattern:
```
$ cd /tmp && git clone https://github.com/hermitdave/FrequencyWords && cd FrequencyWords/content/2018
$ read -r -d '' PYTHONI_CODE <<'EOF'
words = []
seen = set()
for l in stdin:
    parts = l.strip().split(":")
    lang = parts[0].split("/")[0]
    word = parts[-1].split()[0]
    if word not in seen:
        words.append((lang, word))
        seen.add(word)
print("\n".join(f"{word} ({lang})" for lang, word in sorted(words, key=lambda w: (len(w[1]), w[1]))))
EOF
$ rg '.*eac.*' --no-heading | pythoni -c "$PYTHONI_CODE" | head -n 20
eac (pt)
aeac (pt)
beac (bg)
ceac (pt_br)
deac (vi)
eaca (ro)
eacd (es)
eace (en)
each (ur)
eack (en)
eacl (es)
eacn (es)
eaco (pt)
eact (en)
eacă (ro)
eacη (en)
feac (ro)
geac (zh_cn)
ieac (ro)
leac (ru)
$ rg '.*cae.*' --no-heading | pythoni -c "$PYTHONI_CODE" | head -n 20
cae (zh_cn)
acae (gl)
cae' (it)
cae- (es)
cae. (es)
caea (sr)
caeb (pt_br)
caec (es)
caed (zh_cn)
caee (en)
caei (ro)
caej (pl)
cael (es)
caem (pt_br)
caen (zh_cn)
caer (zh_cn)
caes (zh_cn)
caet (pt_br)
caeu (pt_br)
caex (ms)
```

Parse multiple json files found using `fd` to, e.g., sum some metadata
```
fd --full-path '/parent/.*metadata.json' /root/path/to/search | pythoni -c \
<(cat <<'EOF'
import pathlib
import json

total = 0
for line in stdin:
    with pathlib.Path(line.strip()).open("r") as f:
        metadata = json.load(f)
        total += metadata.get("count", 0)

print(f"{total=}")
EOF
)
```

### License
MIT, included at the top of the source file. Go forth and be free to use `pythoni`!
