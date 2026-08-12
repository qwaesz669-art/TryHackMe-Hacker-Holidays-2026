# Try Hack Me: Room 404 (Hacker Holidays — Day 2)


> **_Note:_** _I’m still actively in my learning phase when it comes to web security. My approach might involve a little extra manual snooping or quick-and-dirty scripting, but hey — if it extracts the flag, it works!_

## Challenge Briefing

He booked the quiet room. It’s not on the floor plan, not in the brochure, not on any door. But port 8080 is wide open, and the rooms it never lists are the ones worth finding.

Welcome to the Byte Lotus, where the WiFi is open, the app is free, and the concierge already knows your coffee order. You spend these first days as a guest who simply notices things — a room that isn’t on the floor plan, packets that leave every night at the same hour, a profile assembled from two breakfasts and a livestream.

The Byte Lotus guest-experience platform went live in a hurry, and the night-shift developer shipped more than the website.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*mhAf0x8vhWQNZAT6wWrrdg.png)

---

## 01. Initial Nmap Recon & The Exposed `.git`

After firing up Nmap against the target IP, port `8080` immediately caught my attention. A quick directory scan revealed a classic developer mistake: an exposed repository!

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*aUMsKRw_H1qZAezmhN0qoQ.png)

nmap on target machine IP

[http://10.49.144.21:8080/.git/]

Opening that up in the browser gave us full directory listing access to the internal Git repository. The night-shift developer really was in a rush — they left the entire backend version history public.

---
## 02. Snooping Through Git Objects

![](https://miro.medium.com/v2/resize:fit:593/1*mJ8VTC5FMnVPT4qtR_yoXQ.png)

Navigating through `[http://10.49.144.21:8080/.git/objects/](http://10.49.144.21:8080/.git/objects/)`, I found several directory subfolders containing raw Git object files:

![](https://miro.medium.com/v2/resize:fit:549/1*0AqWN4C0W9ImV5VOG2KCtA.png)

/.git/objects directory

![](https://miro.medium.com/v2/resize:fit:535/1*s_erM9xgt9KCTWGRJOpqGA.png)

I downloaded the loose object files inside these directories. Checking one with the `file` command showed:

![](https://miro.medium.com/v2/resize:fit:549/1*CPmQu6yy9kShwyUY9RlzpQ.png)

Because Git stores object content as zlib-compressed streams, standard `cat` isn't going to help us here.

---
## 03. Decompressing the Artifacts

Instead of manually pulling down full tools, I wrote a quick one-liner Python script in bash to iterate through each object hash and inflate the zlib streams:

for file in 12caa4e52a965e89e5eccf5760924b21aacbf7 13550b4cb13e9f30c61d5b342c532d21e45bda 45dbd69394ea9e13683d9efb6a0220daac59d4 75ab073f67615a27135663ed36794c2d2584fb 965c580fee91d852e5b19a8290da02d2926523; do  
    echo "Decompressing: $file"  
    python3 -c "import zlib; print(zlib.decompress(open('$file', 'rb').read()).decode('utf-8', errors='ignore'))" 2>/dev/null || echo "Failed to decompress."  
done

Running the script printed out the internal source code, staging comments, and…

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*hDwBxQBmtiuM5hfG2Go6-Q.png)

And Hurray! we got the flag!

---
## Key Takeaways

- **Never leave** `**.git**` **exposed on production servers:** Anyone can reconstruct your repository, inspect developer commit histories, and pull sensitive config values.
- **Remove Staging Flags Before Launch:** Leaving hardcoded staging credentials or comments (“remove before launch”) in version control _always_ comes back to haunt you.

