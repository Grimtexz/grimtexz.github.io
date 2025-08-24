+++
title = "To post, or not to post..."
date = "2024-03-28"
tags = ["GoHugo", "HTML", "Python", "Scripting"]
Categories = ["Over My Head", "Learning to blog"]
+++



# Welcome!
---
Hey! Howdy! Guten tag! おはようございます! My name is Kevin Mermelstein. I am an aspiring Cybersecurity Engineer. I am planning on using this blog to showcase my interests, current activities, and general thoughts about a host of various concepts, both related to cybersecurity and not.


## Step 1: Obsidian
---
My first blog post is semi-documentation on my primary workflow process. My text editor for the posts are using Obsidian as a my primary note processor. It being both free and also modifiable helps me design and aesthetically pleasant experience with making markdown documents. I find markdown to not always be the most enjoyable in terms of designing charts or any visualizations, so using Obsidian helps. I will be updating this series with any additional posts that modify this workflow.

### Community plugins I use:

| Plugins           | Developer                                                           |
| ----------------- | ------------------------------------------------------------------- |
| Better Word Count | [Luke Leppan](https://github.com/lukeleppan/better-word-count)      |
| Editing Toolbar   | [PKM-er](https://github.com/PKM-er/obsidian-editing-toolbar)        |
| Iconize           | [Florian Woelki](https://github.com/FlorianWoelki/obsidian-iconize) |
| Kanban            | [Matthew Meyers](https://github.com/mgmeyers/obsidian-kanban)       |
| Markwhen          | [Mark-when](https://github.com/mark-when/obsidian-plugin)           |
| Omnisearch        | [Simon Cambier](https://github.com/scambier/obsidian-omnisearch)    |

I use obsidian for both general notes as well as drafting reports. As this is a tool I use a lot, the idea of simplifying the export of markdown files, from Obsidian to an SSG is a ideal workflow. 

### Documentation Tools

| Program Name | Version  | Purpose                 |
| ------------ | -------- | ----------------------- |
| Flameshot    | 13.0.1   | Screenshoting           |
| OBS          | 31.1.2   | Screen Capture          |
| LibreOffice  | 25.2.4.3 | Process notation        |
| Zotero       | 7.0.22   | Citation and Web scrape |
| VS Code      | 1.103.0  | Scripting               |
| Obsidian     | 1.8.10   | Note Taking             |
| HashMyFiles  | 2.50     | GUI Hashing             |

My process for documenting details, whether it be a forensic report or a cybersecurity documentation. Most of all my tools are FOSS tools. One of my largest repositories of information for me is both Obsidian and Zotero. Obsidian's ability to produce Zettelkasten style note systems and provide note linking to create well-structured mind maps. Zotero is also in my toolbelt, not only to ensure documentation of sources, but also for the value of archiving website data relevant to a specific source. While the same effect can be done using The Way Back Machine, hosting them locally lets me have posterity for my own research.  



## Step 2: Automation Scripting
---
One of my current projects is scripting a python automation script to recognize image file embedding inside a markdown file, It then replaces the embedded wikilink text to a local one. If the attachments are local, they copy it to the correct directory in the static resources folder in a GoHugo site project. It also tries to download images from external URL embeds. It is not a perfect script. Not every website links their images similarly. 

It still has some ways to go to catch all the use cases I want, however, this works for my current uses:

```
###############################################################################
# Script: Image Conversion Remote Image Acquisition                           #
# Version: 1.1.1                                                              #
# Author: Kevin Mermelstein (Grimtexz) + Assistant tweaks                     #
# Date: 2025-08-11                                                            #
# Purpose: Replace Obsidian embeds with web paths and copy/download images    #
#          for Hugo SSG. Refactored using GPT-5. A more comprehensive         #
#          script for automating site production. Inspired by Network Chuck's #
#          blog script.                                                       #
###############################################################################
  
import os
import shutil
import re
from urllib.parse import urlsplit, quote
import requests
import logging

# ------------------ Configuration ------------------

POSTS_DIR = os.getenv('OBSIDIAN_POSTS', r'[Insert Path]')
ATTACHMENTS_DIR = os.getenv('OBSIDIAN_ATTACHMENTS', r"[Insert Path]")
HUGO_STATIC_IMAGES_DIR = os.getenv('HUGO_IMAGE_DIR', r'[Insert Path]')
LOGGING_DIR = os.getenv('LOGGING_DIR', r'.\logs')
os.makedirs(LOGGING_DIR, exist_ok=True)  

# ------------------ Logging ------------------

logging.basicConfig(
	filename=os.path.join(LOGGING_DIR, "ImageConversionScript.log"),
	filemode="w",
	level=logging.INFO,
	format="%(asctime)s - %(levelname)s - %(message)s",
	datefmt="%Y-%m-%d %H:%M:%S"
)

console = logging.StreamHandler()
console.setLevel(logging.INFO)
formatter = logging.Formatter("%(levelname)s: %(message)s")
console.setFormatter(formatter)
logging.getLogger().addHandler(console)
logger = logging.getLogger(__name__)
  

# ------------------ RegEx Patterns ------------------

imageFileTypes = r'(?:avif|bmp|gif|jpe?g|png|svg|webp|tiff?|ico|heic|heif)'

# Single parser for Obsidian-style local wiki-embeds ![[ path/to/file.ext | optional alt ]]

re_local_wikilink = re.compile(
	rf'!\[\[\s*'
	rf'(?P<target>[^\]|#^]+?\.{imageFileTypes})'     # file.ext before | # ^
	rf'\s*(?:[#^][^\]|]*?)?'                         # optional heading/block
	rf'(?:\|(?P<alt>[^\]]*?))?'                      # optional |alt
	rf'\s*\]\]', re.IGNORECASE
)

# Remote embeds: ![alt](https://.../file.ext[?q#f]) with optional <...> and title

re_altText = re.compile(rf'!\[((?:\\.|[^\]])*?)\]\(\s*(?:<\s*)?https?://[^\s"\'()>\]]+?\.{imageFileTypes}', re.IGNORECASE)

re_urls = re.compile(rf'!\[(?:\\.|[^\]])*?\]\(\s*(?:<\s*)?(https?://[^\s"\'()>\]]+?\.{imageFileTypes}(?:\?[^\s"\'()>\]]*)?(?:#[^\s"\'()>\]]*)?)', re.IGNORECASE)

re_url_embed = re.compile(rf'(!\[(?:\\.|[^\]])*?\]\(\s*(?:<\s*)?https?://[^\s"\'()>\]]+?\.{imageFileTypes}(?:\?[^\s"\'()>\]]*)?(?:#[^\s"\'()>\]]*)?(?:\s*>)?(?:\s+(?:"[^"]*"|\'[^\']*\'))?\s*\))', re.IGNORECASE)

# ------------------ Markdown File Walking ------------------
def markdownSearch() -> None:
    for dirpath, _, filenames in os.walk(POSTS_DIR):
        for file in filenames:
            if file.endswith('.md'):
                markdownProcessing(dirpath, file)

# ------------------ Download Remote Images------------------
def downloadRemoteImage(imageURL: str, fileName: str) -> bool:
	try:
		resp = requests.get(imageURL, timeout=15)
		resp.raise_for_status()
		if resp.status_code == 200:
			dest = os.path.join(HUGO_STATIC_IMAGES_DIR, fileName)
			with open(dest, "wb") as f:
				f.write(resp.content)
			logger.info('Downloaded remote image -> %s', imageURL)
			return True
		logger.warning('Unexpected status downloading %s: %s', imageURL, resp.status_code)
		return False
	except Exception as e:
		logger.error("Download failed for %s: %s", imageURL, e)
		return False

# ------------------ Replace Local Embed Values ------------------
def localEmbedsChange(markdownText: str) -> str:
	updated = markdownText
	logger.info('Scanning for local image embeds')
    
	# Iterate over matches from the original text to avoid shifting indices
	for m in re.finditer(re_local_wikilink, markdownText):
		full   = m.group(0)
		target = m.group('target')                   # e.g., "folder/img.png"
		alt    = m.group('alt')
		fname  = os.path.basename(target)            # "img.png"
		if not alt:
			alt = fname
		copied = fileCopy(fname)
		if not copied:
			logger.warning('Skipping embed replacement; source not copied: %s', fname)
			continue
		newEmbed = f"![{alt}](/static/images/{quote(fname)})"
		updated = updated.replace(full, newEmbed)
		logger.info('Local embed converted: %s -> %s', full, newEmbed)
	return updated

# ------------------ File Copy ------------------
def fileCopy(fileName: str) -> bool:
	src_path  = os.path.join(ATTACHMENTS_DIR, fileName)
	dest_path = os.path.join(HUGO_STATIC_IMAGES_DIR, fileName)
	try:
		if not os.path.isfile(src_path):
			logger.warning('Local image not found: %s', src_path)
			return False
        if not os.path.isfile(dest_path):
			shutil.copy2(src_path, dest_path)
			logger.info('Copied local image -> %s', dest_path)
		else:
			logger.info('Local image already present -> %s', dest_path)
		return True
	except Exception as e:
		logger.exception('Failed to copy local image %s: %s', fileName, e)
		return False

# ------------------ Replace Remote Embed Values ------------------
def remoteEmbedsChange(markdownText: str) -> str:
	imageEmbeds = re.findall(re_url_embed, markdownText)
	urlsFound = re.findall(re_urls, markdownText)
	altTexts = re.findall(re_altText, markdownText)
	logger.info('Scanning for remote image embeds')
	updated = markdownText
	for site, text, embed in zip(urlsFound, altTexts, imageEmbeds):
		fileName = os.path.basename(urlsplit(site).path)
		destPath = os.path.join(HUGO_STATIC_IMAGES_DIR, fileName)
		try:
			if os.path.exists(destPath):
				logger.info('Remote image already present -> %s', destPath)
			else:
				ok = downloadRemoteImage(site, fileName)
			if not ok:
					logger.warning('Skipping replacement due to download failure: %s', site)
				continue
		except Exception as e:
			logger.exception('Unexpected error handling %s: %s', site, e)
			continue
		newEmbed = f"![{text}](/static/images/{quote(fileName)})"
		updated = updated.replace(embed, newEmbed)
		logger.info('Remote embed converted: %s -> %s', embed, newEmbed)
	return updated


# ------------------ Markdown File Processing ------------------
def markdownProcessing(path: str, file: str) -> None:
	logger.info('Processing file: %s', file)
	markdownFile = os.path.join(path, file)
	with open(markdownFile, 'r', encoding='utf-8') as f:
		content = f.read()
		content = localEmbedsChange(content)
		content = remoteEmbedsChange(content)
	with open(markdownFile, 'w', encoding='utf-8', newline='') as f:
		f.write(content)
	logger.info('Finished file: %s', file)

# ------------------ Main ------------------
def main() -> None:
	markdownSearch()

if __name__ == '__main__':
	main()
```

Key area's I will be focusing on moving forward is the RegEx Patterns. I am not that great with RegEx just yet. I understand it conceptually, but the mechanical logic can feel bogged down to me. I am chocking that up to experience at this moment in time. Additionally, I am considering using PIL library in order to convert the binary content from the HTTP request, and automatically convert the image type to jpg. 



## Step 3: GoHugo SSG
---
SSGs (Static Site Generators)... 

***Exhaling...*** 

Not to go into a diatribe, but there are too many to choose from. Jekyll, Next.js, Docusaurus, Nuxt, Gatsby, Astro among the many... They all roughly do the same thing. Converting a simpler format like markdown and converting it into an HTML format based on either pre-done or user-defined themes. The main differences are what is used to achieve a static site.

My choice for using GoHugo is slightly arbitrary. Jekyll would have also have been a perfectly reasonable choice for creating a Github Pages site. For me, the biggest factor is my lack of familiarity with Ruby and the simplicity of Hugo's conversion process and theme acquisition.

My choice for using an SSG is specific to three main factors:
1. I do not like Common CMS services like Word Press. The sites feel identical. And if I am going to put all that effort in, I want something unique. 
2. While I do understand HTML and JS, I do not code in them. It is not something I do on a common basis. I would rather my energy be spent on generating content rather than hand coding a website.
3. I do not like the security options on CMS sites.



## Step 4: Plans moving forward
---
TBD

