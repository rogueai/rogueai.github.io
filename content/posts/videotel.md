+++ 
title = "`DRAFT` Videotel SIP TTM90"
date = 2022-02-11 
description = "Use a Videotel device as a dumb terminal"
draft = false 
toc = false 
author = "rogueai"
categories = ["post"]
tags = ["videotel"]
license = "cc-by-nc-sa-4.0"
+++

# Background

In the late 80s through early 90s, Videotel was the Italian equivalent of the French Minitel: a terminal that was able
to access Teletext services, provided under a subscription fee.

While in France the Minitel got quite popular, with services still active in the early 2000s, in Italy it never got to
the same level of adoption, both for the subscription being quite expensive and a number of legal issues that arose
among the companies involved.

Although the original teletext services have long died, a Videotel can still be used as a dumb terminal when connected
to a Linux box.

# Concept
The gist of this little project is to connect the Videotel to a linux box via serial port.

> Note: I'll be using the Videotel model `SIP TTM90/TM`. There are two versions of this model: the `/TM` version has a
> serial port on the back, while the non-`/TM` version does not. 
> 
> Apart from the obvious missing port, the two models also have a cosmetic difference: the front button being grey on the
> model with a serial port, and red on the other one.


