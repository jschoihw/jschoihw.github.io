---
title: "PortSwigger Academy JWT Labs Walkthrough"
date: 2025-08-13 00:00:00 +0800
categories: [Web, PortSwigger]
tags: [portswigger, web-security, writeup, jwt]
---


## 📜 Introduction


---

## 🔹 Lab 1 – JWT authentication bypass via unverified signature

### Description

This lab uses a JWT-based mechanism for handling sessions. Due to implementation flaws, the server doesn't verify the signature of any JWTs that it receives.

 To solve the lab, modify your session token to gain access to the admin panel at /admin, then delete the user carlos.

 You can log in to your own account using the following credentials: wiener:peter


### Solution
---

First, log in with the credentials `wiener:peter`. Attempt to access `/admin`. You’ll receive an error message denying access: 
![Error Message](/assets/img/2025-08-14_20-03.png)

Open **Burp → Proxy → HTTP history**. If the Burp extension **JWT Editor** is installed, you will see requests containing JWTs highlighted in green: 
![JWT](/assets/img/2025-08-14_20-07.png)

Send one of these requests to **Repeater**. In the “JSON Web Token” tab, Burp automatically decodes the token, allowing you to edit claims or perform built-in JWT attacks.  

Since this application does **not verify JWT signatures**, you can directly modify the `"sub"` claim in the payload to `"administrator"`: 
![Editing JWT](/assets/img/2025-08-14_20-13.png)

Burp automatically updates the `Cookie` value with the modified JWT. Send this modified request to `/admin`. Then, the server responds with `200 OK` and reveals the admin interface and the request which enables the admin to delete the users:

![Deleting Carlos](/assets/img/2025-08-14_20-18.png)

Finally, trigger the deletion by sending a request to `/admin/delete?username=carlos` with the modified JWT and the challenge is complete.