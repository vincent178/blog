---
title: "Command"
date: 2020-03-10T13:48:23+08:00
draft: true
---

// command 允许任意个数的参数
-nargs=* 


that arguments are used as text, not as expressions.




prices_by_condition = active_products.group(:shoe_condition).minimum(:price_cents)

