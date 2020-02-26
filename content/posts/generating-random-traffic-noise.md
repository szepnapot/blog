---
title: "Generating random HTTPS/DNS traffic noise"
date: 2019-02-02T15:41:54+01:00
description: "Adding the top 1 million Alexa sites and fine tune the scraping attributes"
categories: [ "bash", "python", "unix", "shell" ]
tags: ["bash", "unix", "shell", "linux"]
---

### Intro

I've recently came across a nice python project on github called [noisy](https://github.com/1tayH/noisy). It says:

>A simple python script that generates random HTTP/DNS traffic noise in the background while you go about your regular web browsing

The setup is pretty straightforward, the config file however needs some polishing. The default values are quite basic so I created a small bash script to set it up for me properly.

### Tweaking

The script downloads the current*ish* (<2 weeks) Alexa top 1 million list from S3, then post-process it using Python. We simply clean the downloaded csv and update noisy's config file.

```bash
#!/usr/bin/env bash
set -o errexit
set -o pipefail
set -o nounset

git clone https://github.com/1tayH/noisy.git && cd noisy

readonly __dir="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

curl -sSL "http://s3.amazonaws.com/alexa-static/top-1m.csv.zip" | tar xvfz - -C ${__dir}

python - <<EOF
import json
import csv
import os

# load noisy's config
config_file = 'config.json'
top_1m = "top-1m.csv"
with open(config_file) as f:
    config = json.load(f)

# add top sites and modify depth + sleep
with open(top_1m) as csv_file:
    csv_reader = csv.reader(csv_file, delimiter=',')
    for row in csv_reader:
        config["root_urls"].append("https://{}".format(row[-1]))

try:
    os.remove(top_1m)
except OSError:
    pass

config["max_depth"] = 15
config["min_sleep"] = 1
config["max_sleep"] = 3

# update noisy's config
with open(config_file, 'w') as json_file:
    json.dump(config, json_file)
EOF
# back to bash
echo "[*] updated max_depth -> 15 | min_sleep -> 1 | max_sleep -> 3"
echo "[*] Alexa top 1 million site added to noisy root urls"
echo "python noisy.py --help"
echo "[*] example: python noisy.py --config config.json"
``` 

### Usage

Save the above snippet to `setup_noisy.py`.

Then run `bash setup_noisy.py`.

I found it useful during pentesting, but you definitely can make harder ad companies life if you ran this in the background or even if you (like myself), owns a Raspberry Pi, which can function as a [home Adblocker](https://pi-hole.net/).
On this Pi-Hole I also set up noisy to run constantly in the background just to test it out.
With `nohup` it's quite simple `nohup python noisy.py --config config.json </dev/null > noisy.log 2>&1 &`

### Summary

I really like this block when I write bash scripts.

```bash
python - <<EOF
[PYTHON SCRIPT]
EOF
```

and this one too

```bash
curl -sSL "URL" | tar xvfz - -C [PATH TO EXTRACT TO] 
```
