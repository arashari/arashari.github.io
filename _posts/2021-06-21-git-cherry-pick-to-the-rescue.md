---
layout: post
title: <code>git cherry pick</code> to the rescue
tags: [git, personal exp, trick]
---

jadi beberapa waktu lalu saya diminta untuk mengerjakan suatu fitur, jadilah saya develop itu dari branch `master` seperti biasa. tapi ternyata eh ternyata, sayang sekali khusus untuk fitur tersebut, seharusnya didevelop dari branch khusus.

jadilah saya harus memindahkan kodingan yang sudah dikerjakan ke branch baru.

sedih :(

_well_, mungkin pada saat itu karena sudah malam dan lelah, jadinya tanpa pikir panjang saya pindahkan semua kodingan yang sudah dikerjakan secara *MANUAL*.

---

beberapa hari berikutnya, ternyata ada kebutuhan untuk memindahkan kodingan lagi.

_oh no_

untungnya sekarang sudah bisa berpikir jernih
> _"ah masa ga ada cara yang elegan untuk kasus seperti ini"_

_googling-googling_ jadilah ketemu command `git cherry-pick` yang cocok untuk kasus ini
```
git cherry-pick -n <commitid>
```

:)

---

- [https://www.devroom.io/2010/06/10/cherry-picking-specific-commits-from-another-branch/](https://www.devroom.io/2010/06/10/cherry-picking-specific-commits-from-another-branch/) 
