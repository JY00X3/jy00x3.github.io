---
title: "The Nested Archive Nightmare — A Four-Layer OSINT Deep Dive"
date: 2025-11-30
categories: [CTF, OSINT]
tags: [osint, geolocation, wikipedia, md5, nested-archives]
image:
  path: /assets/nasted_archive/4.png
---

This **OSINT challenge** was pure evil—four layers of nested 7z archives, each locked behind an **MD5-hashed Wikipedia URL** with only a cryptic photo per layer to guide you. No hints. No mercy. Just rage-fueled searches at 3 a.m.

In this post, I'll walk through how I survived this four-stage nightmare.

---

## 📌 Challenge Overview

**Difficulty:** Hard  
**Category:** OSINT  
**Author:** Unknown Sadist  

![Desktop View](/assets/nasted_archive/1.png){: width="700" height="400" .normal }

![Desktop View](/assets/nasted_archive/2.png){: width="700" height="400" .normal }

The challenge structure:

- **Stage 0 → Stage 1:** Unlock with given password → Extract 1.7z + 1.png  
- **Stage 1 → Stage 2:** Identify location in photo → Hash Wikipedia URL → Get password  
- **Stage 2 → Stage 3:** Repeat process  
- **Stage 3 → Stage 4:** Retrieve final flag  

Each layer required precise **geolocation**, exhaustive **Wikipedia URL hunting**, and enough **mental stamina** to survive the endless dead ends.

---

# 🟩 Stage 0 → Stage 1 — The Creepy Garden

**Given Archive:** 0.7z  
**Password:** CyberTalents  
**Contents:** 1.7z + 1.png  
![Desktop View](/assets/nasted_archive/3.png){: width="700" height="400" .normal }


## 🔍 Image Analysis

![Desktop View](/assets/nasted_archive/4.png){: width="700" height="400" .normal }


At first glance: a serene garden with haystacks, flowers, and what appears to be a yellow-and-white striped balloon floating overhead.

![Desktop View](/assets/nasted_archive/5.png){: width="700" height="400" .normal }

**Zoom in and the nightmare begins:**

That "balloon" has a face. It's a life-sized doll suspended on a wire—straight out of a horror film.

![Desktop View](/assets/nasted_archive/6.png){: width="700" height="400" .normal }


## 🧭 The Hunt Begins

Initial searches went nowhere:

- "Floating dummy garden" ❌  
- "Striped balloon field" ❌  
- "Japanese scarecrow doll" ❌  

After **45 minutes of frustration**, the pattern finally clicked:

This is **Nagoro Doll Village** in Tokushima Prefecture, Shikoku, Japan—a dying rural town where locals repopulated the area by crafting life-sized dolls as scarecrows.

![Desktop View](/assets/nasted_archive/7.png){: width="700" height="400" .normal }

![Desktop View](/assets/nasted_archive/8.png){: width="700" height="400" .normal }

![Desktop View](/assets/nasted_archive/9.png){: width="700" height="400" .normal }

## 🔐 The Wikipedia Gauntlet

Now came the real torture: finding the exact Wikipedia URL whose MD5 hash would unlock the next archive.

Tried:

- "Nagoro Scarecrow Village" ❌  
- "Kakashi no Sato" ❌  
- Various Japanese titles ❌  

![Desktop View](/assets/nasted_archive/10.png){: width="700" height="400" .normal }

**The winning URL:**

```shell
https://en.wikipedia.org/wiki/Nagoro
```

**MD5 Hash:**

```shell
d34af31c6f95e8db2b3aa451a8f9d63a
```

✅ **Archive unlocked.**

---

# 🟩 Stage 1 → Stage 2 — The Castle Hunt

**Contents of 1.7z:** 2.7z + 2.png  
![Desktop View](/assets/nasted_archive/11.png){: width="700" height="400" .normal }


## 🔍 Image Analysis

Rolling hills. Distant mountains. Picturesque scenery. Too easy, right?

![Desktop View](/assets/nasted_archive/12.png){: width="700" height="400" .normal }


**Wrong.**

After 10 agonizing minutes of pixel-by-pixel inspection, I spotted it—a **minuscule castle** tucked on the far-right hill, barely visible without magnification.

![Desktop View](/assets/nasted_archive/13.png){: width="700" height="400" .normal }


## 🧭 Geolocation Deep Dive

The search launched across:

- "Small castles Europe" ❌  
- "Hilltop ruins Italy" ❌  
- "Medieval fort Abruzzo" ❌  

Finally cracked it: **Castello Caldora** in Pacentro, Italy—a 13th-century fortress nestled in the Abruzzo mountains.

![Desktop View](/assets/nasted_archive/14.png){: width="700" height="400" .normal }


## 🔐 Wikipedia Round Two

More trial-and-error:

- "Pacentro" ❌  
- "Italian castles" ❌  

**The exact URL that worked:**

```shell
https://en.wikipedia.org/wiki/Castello_Caldora
```

**MD5 Hash:**

```shell
879a7a1efb80cf7b5a00b5eb5ca290b7
```

✅ **Next archive opened.**

---

# 🟩 Stage 2 → Stage 3 — The Thatched Village

**Contents of 2.7z:** 3.7z + 3.png  

![Desktop View](/assets/nasted_archive/15.png){: width="700" height="400" .normal }

## 🔍 Image Analysis

![Desktop View](/assets/nasted_archive/16.png){: width="700" height="400" .normal }

A dimly lit interior view overlooking bamboo stacks and traditional thatched-roof homes—the architecture screams **traditional Japanese gassho-zukuri construction**.

![Desktop View](/assets/nasted_archive/17.png){: width="700" height="400" .normal }



## 🧭 The Killer Stage

This was brutal. Japan has *dozens* of these villages. I exhaustively searched:

- "Bamboo traditional Japan homes" ❌  
- "Thatched roof villages Shikoku" ❌  
- "UNESCO heritage sites Japan" ❌  

Cross-referenced against known locations:

- Shirakawa-go  
- Gokayama  
- Ogimachi  
- Ainokura  

**30 minutes later:** The architectural details matched **Shirakawa village area** in Gifu Prefecture—steep roofs, bamboo arrangement, traditional construction methods.

![Desktop View](/assets/nasted_archive/18.png){: width="700" height="400" .normal }


## 🪤 The Disambiguation Page Trap

![Desktop View](/assets/nasted_archive/19.png){: width="700" height="400" .normal }

Here's where the challenge designer got clever. The *obvious* Wikipedia pages failed:

- "Shirakawa-go" ❌  
- "Historic Villages of Shirakawa-go" ❌  

The solution? An **obscure disambiguation page:**


```shell
https://en.wikipedia.org/wiki/Shirakawa,_Gifu_(village)
```

**MD5 Hash:**

```shell
fa94afd96d3836abbdae2bbde08ba3c0
```

✅ **Archive unlocked. Final stage revealed.**

---

# 🚩 Final Stage — The Flag

After unlocking the final 3.7z archive, a single text file appeared containing:
![Desktop View](/assets/nasted_archive/20.png){: width="700" height="400" .normal }


```shell
flag{e827a89870a9089870ca28d28e50ceb7}
```

---

## 🧠 Takeaways

This challenge was a masterclass in **applied OSINT fundamentals**:

- **Geolocation precision:** Spotting tiny details (a doll's face, a distant castle, architectural styles)  
- **Exhaustive searching:** Trying dozens of search term variations when initial guesses fail  
- **Wikipedia URL hunting:** Understanding that exact titles matter—and sometimes obscure disambiguation pages are the key  
- **Cross-validation:** Using multiple sources (Google Images, satellite maps, architectural databases) to confirm locations  
- **Persistence:** Surviving 3+ hours of rage-induced searching without losing your mind  

The real challenge wasn't identifying the locations—it was finding the *exact* Wikipedia URL format the author chose. That's the trap, and it's brilliant.

# Happy hacking 🧑‍💻🌍

---

