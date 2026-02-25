---
layout: post
title: How to connect Personal Linux device to the Managed Print Service (University of Reading)
categories: [Linux, UoReading]
description: Use University of Reading's Printers on Linux
keywords: UoReading, Linux
lang: en
---

Here is a tutorial of using the University of Reading's FollowMe Printer on Linux.

# Pre-requisites

This section is copied from the [official instruction for Windows devices](https://www.reading.ac.uk/digital-technology-services/it-help-and-support/how-to-connect-to-the-managed-print-service/how-to-connect-personal-windows-device-to-the-managed-print).

- Your device must be connected via Eduroam to the University of Reading network (or wired connection).
- You must not be using a VPN
- You must have a valid University of Reading IT user account. This service is unavailable to visitors. 


# Instructions

Firstly, install `cups` and `samba`, and other necessary packages. Please refer to [Arch Wiki](https://wiki.archlinux.org/title/CUPS) or the document of your own Linux distro. 

Once `cups` and `samba` are configured properly, visit [`http://localhost:631`](http://localhost:631) in your browser. If a popup asks for your username and password, use the ones of your Linux system. Then:

1. Click `Administration` in the top bar. 
2. Click `Add Printer` in the "Printer" section. 
3. Choose `Windows Printer via SAMBA`. 
4. Enter the following address in the `Connection` field:
`smb://RDG-HOME/<your_student_number>:<your_password>@uorprint.rdg.ac.uk/FollowMe-BW`
The student number is like "ab123456", and special characters in your password (if present) should be escaped following the [URL encoding rule](https://en.wikipedia.org/wiki/Percent-encoding#:~:text=Reserved%20characters%20after%20percent%2Dencoding). 
5. Click `Continue`.
6. Give it a name, for example, `FollowMe-BW`.
7. Choose `Generic` for `Make`, and `Generic PostScript Printer` for `Model`.
8. Save.
9. Add the `FollowMe-Colour` for colour printing if you need it.
10.  Congratulations! Now you can send your files to the University Printers.

# Notes

## Security concerns

Your username and password will be stored in plaintext in `/etc/cups/printers.conf`. However, this file can only be read by its owners (in my case, `root` and `cpus`), so the risk is relatively small for personal computers. 

If you cannot accept this risk, remove your username and password from the URL and use `smb://uorprint.rdg.ac.uk/FollowMe-BW`. Theoretically, in this case, you will be asked for your username and password when you create a print job. I'm not sure whether any desktop environment offers a "remember password" feature. 

## My credential is hardcoded but still being asked

run this command in your console:

`sudo lpadmin -p FollowMe-BW -o auth-info-required=none`
