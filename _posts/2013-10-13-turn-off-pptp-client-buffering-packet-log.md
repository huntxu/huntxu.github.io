---
layout: post
title: 關掉 pptp client buffering packet 的 log
---

由于某些原因 pptp client 老是報 buffering packet，而且頻繁發生，導致 journalctl 一看的話全都是 pptp 的信息。搜了一圈沒發現有人提供這個問題的解決辦法，于是自己找 pptp 的 manpages，發現了一個叫 --loglevel 的參數。于是打開 /etc/ppp/peers/PROVIDER，把 "--loglevel 0" 加到 pty 選項那句 pptp 的啟動命令之後，從此世界清淨了。
