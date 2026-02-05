---
title: "PatriotCTF 2025 — Where’s Legally Distinct Waldo (OSINT Challenge 3)"
date: 2025-11-30
categories: [CTF, OSINT]
tags: [osint, geolocation, patriotctf, gmu]
image:
  path: /assets/patrioctf/image.png
---
---

The **PatriotCTF 2025** roster featured a fun, four-part **OSINT** challenge track titled  
**“Where’s Legally Distinct Waldo.”**

This series was a true **geolocation scavenger hunt**, pushing players to precisely identify **highly specific locations** hidden across the **George Mason University (GMU)** campus.

![Desktop View](/assets/patrioctf/image2.png){: width="700" height="400" .normal }

In this post, I’ll walk through how I solved:

# 🟩 Challenge 3 — Where’s Legally Distinct Waldo Three

---

## 📌 Challenge Prompt

![Desktop View](/assets/patrioctf/image3.png){: width="700" height="400" .normal }


**Image Given:**

![Desktop View](/assets/patrioctf/image4.png){: width="700" height="400" .normal }

---

## 🧭 Step 1 — Campus Identification

After careful visual analysis, the image was **confidently placed on the George Mason University campus**. This immediately narrowed the search space from a general geolocation problem to a **targeted reconnaissance task** focused solely on GMU landmarks.

![Desktop View](/assets/patrioctf/image5.png){: width="700" height="400" .normal }

---

## 🌊 Step 2 — Recognizing Mason Pond

One distinctive environmental feature stood out:

![Desktop View](/assets/patrioctf/image6.png){: width="700" height="400" .normal }


That’s **Mason Pond**, a well-known landmark on campus. Identifying it helped establish orientation and viewing direction.

---

## 🅿️ Step 3 — Environmental Correlation

Another critical detail in the image was a **large parking lot**:

![Desktop View](/assets/patrioctf/image7.png){: width="700" height="400" .normal }


To validate the hypothesis, I cross-referenced the scene with **Google Maps and satellite imagery**.

![Desktop View](/assets/patrioctf/image8.png){: width="700" height="400" .normal }


Matching indicators included:

- Parking lot layout  
- Tree placement  
- Sidewalk paths  
- A small bridge structure  

This confirmed the **exact vantage point**.

---

## 🏛️ Step 4 — Building Identification

The view aligned directly with a specific building:

![Desktop View](/assets/patrioctf/image9.png){: width="700" height="400" .normal }


Switching to **Street View** allowed me to verify the building’s name by spotting the signage:

![Desktop View](/assets/patrioctf/image10.png){: width="700" height="400" .normal }

**Center for the Arts / Concert Hall**

---

## 🚩 Final Flag

After a few formatting attempts, the accepted flag was:
```shell
pctf{Center_for_the_Arts_Concert_Hall}
```

---

## 🧠 Takeaway

This challenge was a great demonstration of **OSINT fundamentals**:

- Landmark recognition  
- Environmental correlation  
- Satellite + Street View cross-validation  

Small details win geolocation challenges — especially when the hunt is *legally distinct* 😉

Happy hacking 🧑‍💻🌍
