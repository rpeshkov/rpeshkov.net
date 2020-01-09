---
title: "Update timestamp in Hugo post"
date: 2019-02-02T19:02:35Z
tags: ["hugo"]
---

This site is built entirely with Hugo. Hugo itself has a lot of useful commands bundled, but there's one case which is missing for me: update timestamp of some particular post without editing it manually.

My usecase for this is that when I generate new post, I might be writing in more than 1 day because of different reasons. But when I finish the post and want to publish it, I want post to have the date when it was finished. In that case I need to open the post, go to preamble and change the date manually.

Since, I'm quite lazy, I decided to automate this thing and created small Python script that work similar to `touch` command in \*nix systems. This script goes through file and tries to find `date` field in preamble. If it was found, it just updates the field with current date. Very handy!

Here's the script:

```python
#!/usr/bin/env python3

import sys
from datetime import datetime

if len(sys.argv) < 2:
    print('Please provide filename')
    sys.exit(1)

with open(sys.argv[1], 'r') as f:
    lines = f.readlines()

found = False
outlines = []

now_date = ''

for line in lines:
    if not found and line.startswith('date'):
        now_date = datetime.utcnow().strftime("%Y-%m-%dT%H:%M:%SZ")
        line = 'date: ' + now_date + '\n'
        found = True

    outlines.append(line)

if not found:
    print('Date was not found in file!')
    sys.exit(1)

with open(sys.argv[1], 'w') as f:
    f.writelines(outlines)

print('Date updated to ' + now_date)
```

It's really simple and works only with `YAML` preamble, but it do the job for me. I placed it into `/usr/local/bin` and now, when I need to update date to current in any of my blog posts, I just simply invoke it like

```bash
hugo-touch ./content/glob/update-timestamp-hugo-post.md
```

Unfortunately, I haven't found the way to extend Hugo itself with additional command, like it's possible to do with Git. If you know how to do this, please drop me a message in Twitter.
