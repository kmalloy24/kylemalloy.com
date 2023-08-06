---
title: "Writing AutoLISP With ChatGPT"
description: "Writing AutoLISP With ChatGPT"
slug: "LISP-GPT"
date: 2023-04-25
tags: ["chatgpt", "lisp", "autocad"]
---

## Prompt
Write code for autocad lisp command for total length 

## Code Output
```lisp
(defun c:totallength (/ ss len)
  (setq ss (ssget))
  (setq len 0.0)
  (if ss
    (progn
      (setq nents (sslength ss))
      (repeat nents
        (setq ent (ssname ss (setq i (1- i))))
        (setq len (+ len (vla-get-length 
        (vlax-ename->vla-object ent))))
      )
      (princ (strcat "\nTotal length: " (rtos len)))
    )
    (princ "\nNo objects selected.")
  )
  (princ)
)
```
## Instructions from ChatGPT
This command defines a new function called `totallength` and then defines two variables: `ss` to store the selection set and `len` to store the total length of the selected objects.

The function first uses the `ssget` function to prompt the user to select objects. If objects are selected, it then loops through each object in the selection set and adds the length of each object to the `len` variable. Finally, it prints the total length of the selected objects using the `princ` function.

To use this command, simply load the Lisp file in AutoCAD and type `totallength` at the command prompt.