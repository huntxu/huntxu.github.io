---
layout: post
title: 用 find 和 ptags 生成 python 代碼的 tags
---

C 的項目有 ctags，python 項目有 ptags。

工作中遇到有 python 的項目，每次都自己寫整行的 find，還經常把正則寫錯，>.<

于是今天受不了了，直接寫成一個 bash 的 function 好了...

    [hunt@psycho ~]$ O_< type pytags
    pytags is a function
    pytags ()
    {
            find $1 -type f -regex ".*\.py\(\.in\)?" -exec /usr/lib/python2.7/Tools/scripts/ptags.py {} \+
    }

用法倒簡單，我都是直接到代碼目錄下運行 "pytags ." 搞掂...
