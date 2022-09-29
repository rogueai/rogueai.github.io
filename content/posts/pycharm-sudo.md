+++
title = "PyCharm: Debug script or module with `sudo` privileges"
date = 2022-09-29
description = "A quick rundown on how to debug a python module to be ran with `sudo` in PyCharm"
draft = false
toc = true
author = "rogueai"
categories = ["post"]
tags = ["python", "pycharm", "debug"]
license = "cc-by-nc-sa-4.0"
+++

I was trying to debug a python module using PyCharm, however the module requires sudo to run and
it seems it's not possible to set a custom python command to be run in the default interpreter
configuration.

I tried following different approaches (see: [here](https://intellij-support.jetbrains.com/hc/en-us/community/posts/206587695-How-to-run-debug-programs-with-super-user-privileges?page=1#community_comment_205675625) 
and [here](https://esmithy.net/2015/05/05/rundebug-as-root-in-pycharm/)), but they were either 
not working, or just cumbersome to setup.

> Disclaimer: this approach requires PyCharm Professional

We'll be using a local  Python Debug Server for this approach

- Create a new Python remote debug server configuration[^1]
- Leave `localhost` as the *IDE host name*
- Set a port, e.g.: `2345`
- Install `pydevd-pycharm`: `pip install pydevd-pycharm~=<version of PyCharm on the local machine>`
  You can find your PyCharm's version from *Help -> About*, it's something like `#PY-<version>`. The install command can
  also be copied from the remote server configuration dialog.
- Add this to your main (can be copied from the remote server configuration dialog as well):
  ```python
  pydevd_pycharm.settrace('localhost', port=2345, stdoutToServer=True, stderrToServer=True)
  ```
- Start the debug server and set breakpoints
- From a terminal, start your script with `sudo`
  ```bash
  sudo python -m your_module
  ```

That's it!

[^1] https://www.jetbrains.com/help/pycharm/remote-debugging-with-product.html#remote-debug-config