---
# MARP
theme: default
paginate: true

# Jekyll
title: "1_Getting Started"
date: 2023-04-01
categories:
  - Guides
---

*OpenGL programming in Python requires a little effort to set up the development environment. This post tells you how.*

# Software Requirements

We will be building a computer graphics framework in Python with the OpenGL library. For this project, we need to have installed the following software:

- Python 3.7 or higher
- PyGame
- Numpy
- PyOpenGL
- PyOpenGL-accelerate

This guide will show you how to set up and confirm your development environment now so you do not have any surprises later during the project.

In particular, I recommend installing everything you need in a virtual environment which you can turn on and off as needed. Then the software configurations for this project will not interfere with other projects you may have on the same machine.

This guide will explain everything about setting up and using the virtual environment in addition to installing the required packages.

:warning:**&nbsp;This guide uses bash commands in the terminal.&nbsp;**:warning:

On **MacOS** these commands should all work normally in your standard terminal or the one provided in **[VSCode](https://code.visualstudio.com/)**.

On **Windows**, I recommend either installing a bash terminal such as **Git Bash** which comes with **[Git for Windows](https://gitforwindows.org/)**, or use the terminal in **[VSCode](https://code.visualstudio.com/)**.

# Python 3.7

First, make sure you have Python 3.7 or higher. Open your terminal and type `python --version`. You should see something like this:

```bash
Python 3.7.3
```

If the version number is below 3.7, try the following the instructions below for your operating system.

## MacOS

On **MacOS**, the version might be 2.7. In that case, try `python3 --version` to see if you also have Python 3 installed separately. If not, it is better to install Python 3 which can easily be done with Homebrew. I highly recommend installing **[Homebrew](https://brew.sh/)** if you do not have it already. Then use Homebrew to install Python 3:

```bash
$ brew install python3
```

Once the installation completes, you can confirm the version with `python3 --version` again.

## Windows

On **Windows**, you can upgrade Python using an installer from the [Python Downloads](https://www.python.org/downloads/windows/) page. Just open the page and click the "Latest Python 3 Release" link.

If you get the option during installation, be sure add Python to your `PATH` variable. 

After the installation completes, open a terminal and confirm the version number with `python --version` again. 

# Virtual Environment

Once we have Python 3 installed, we can create a virtual environment using its built-in `venv` module. According to the [docs](https://docs.python.org/3/library/venv.html), `venv` is a simple virtual environment tool which provides the following benefit:

> Each virtual environment has its own Python binary (which matches the version of the binary that was used to create this environment) and can have its own independent set of installed Python packages in its site directories.

This means we can create an isolated installation of Python with only the necessary packages and we don't need to worry about breaking other projects or software on our PC.

First, make a directory somewhere on your machine for this project and navigate to it in your terminal. Then type the following command:
 
```bash
$ python3 -m venv .venv
```

This will use the `venv` module to create a virtual environment with your Python 3 binaries in a folder called `.venv`. I like to name the environment folder with a dot (`.`) so it is easy to distinguish and always appears at the top of directory listings. This also hides the folder by default when viewing files in programs like Finder on MacOS or File Explorer on Windows.

Before we can start using the virtual environment, we must be activate it. Fortunately, the environment we just created provides a binary script in its `bin` folder to do just that. We can run the script with the following command in the same place where you created the `.venv` folder:

**MacOS**
```bash
$ source .venv/bin/activate
```

**Windows**
```bash
$ source .venv/Scripts/activate
```

:warning:**&nbsp;Notice the space and dot `.` after `source`&nbsp;**:warning:

Now you should see the name of your virtual environment in parentheses at the front of your command prompt. That means you are working inside the active virtual environment and any Python modules you install now will be exclusive to that environment. If you try the `pip list` command now, you will probably see that only the bare minimum packages are installed:

```bash
(.venv) $ pip list
Package    Version
---------- -------
pip        19.0.3
setuptools 40.8.0
```

With `.venv` we have a clean, minimal environment in which we can freshly install only the things we need.

Whenever you finish working in the environment, always remember to deactivate it so you don't accidentally use it on another project. You can do this simply by typing `deactivate` in the terminal.

```bash
(.venv) $ deactivate
```

:warning:**&nbsp;Always `activate` before you start and `deactivate` when you stop working.&nbsp;**:warning:

# Python Packages

Additional software packages can be installed with `pip` which automatically finds the necessary source, downloads them, and installs them. However, `pip` needs some manual configuration of its network settings if you are using it behind a proxy server, such as on the KIT school network.

## `pip` Proxy Configuration

The `pip list -o` command gets the latest versions of your installed packages from package repositories. We can use this command to test whether `pip` can connect or not. Try typing `pip list -o` into your terminal. If it gives an error, then you might need to configure proxy settings for `pip` to work correctly.

Here we will configure `pip` to use the KIT internal network proxy server only when our virtual environment is active.

First, activate the virtual environment as mentioned above and then type the following command:

```bash
(.venv) $ pip config --site set global.proxy http://wwwproxy.kanazawa-it.ac.jp:8080
```

You should see a message that says it is writing to a `pip.conf` file (for **MacOS**) or a `pip.ini` file (for **Windows**). Make sure the file is inside your virtual environment `.venv` directory.

If this doesn't work, you can create a blank `pip.conf` or `pip.ini` file and write these contents to it:
```
[global]
proxy = http://wwwproxy.kanazawa-it.ac.jp:8080
```

Finally, confirm that the proxy is set with the following command in your terminal and you should see the proxy server address echoed back to you.

```bash
(.venv) $ pip config get global.proxy
http://wwwproxy.kanazawa-it.ac.jp:8080
```

And confirm that `pip` can communicate through the proxy with the `pip list -o` command again. If you don't get an error and see a list of package names and version numbers, then you are ready to install packages!

By the way, you should disable the proxy setting when you need to use `pip` outside the KIT network. The `unset` command can remove the proxy setting:

```bash
(.venv) $ pip config --site unset global.proxy
```

Of course, deactivating the virtual environment will also disable the proxy and any packages you install will be available everywhere on your computer.

## Installing Packages

It is always good to use the latest version of `pip` to make sure everything is up-to-date, so let's upgrade it first. With your `.venv` environment active, run the command:

```bash
(.venv) $ python -m pip install --upgrade pip
```

:fire: Notice we are not using the `python3` command anymore. We created the `.venv` virtual environment with Python 3, so the environment only has Python 3 installed and we can use the `python` command without worrying which version it will use. :sunglasses:

You will see a message that says `Successfully installed pip-XX.XX` when the upgrade completes.

Now we can install the required packages:
```bash
(.venv) $ pip install pygame numpy pyopengl pyopengl-accelerate
```

When installing on **Windows**, you might see an error that says `Microsoft Visual C++ 14.0 is required`. If that happens, try the instructions below for [Troubleshooting on Windows](#troubleshooting-on-windows) before continuing.

The latest versions of each package for the installed Python version should be enough. When the installation completes, let's confirm they are usable in a Python program. Run the interactive Python interpreter by typing `python` in the terminal and pressing Enter. The prompt will change into `>>>` and you can directly enter Python script. 

```bash
(.venv) $ python
Python 3.7.3 (v3.7.3:ef4ec6ed12)
Type "help", "copyright", "credits" or "license" for more information.
>>> █
```
:arrow_right: With **Git Bash** on **Windows**, the terminal might freeze after typing `python` and never respond. If that happens, look under the [Troubleshooting on Windows](#troubleshooting-on-windows) section below for help.

Try importing the installed packages to confirm their installation. If you do not see any errors, then you are all ready to begin the project!

```bash
>>> import pygame
pygame 2.1.2 (SDL 2.0.18, Python 3.7.3)
Hello from the pygame community. https://www.pygame.org/contribute.html
>>> import numpy
>>> import OpenGL
>>> import OpenGL.GL
>>> exit()
(.venv) $ █
```

:warning:**&nbsp;In Python, the module is `OpenGL` not `opengl`.&nbsp;**:warning:

On **MacOS**, you might get an error from `import OpenGL.GL`. In that case, try the instructions for [Troubleshooting on MacOS](#troubleshooting-on-macos) below.

## Troubleshooting on Windows

When installing PyOpenGL and PyOpenGL-accelerate on Windows, you may encounter this error: 

> Microsoft Visual C++ 14.0 is required. Get it with "Build Tools for Visual Studio": https://visualstudio.microsoft.com/downloads/

If that happens, open the [Visual Studio Downloads](https://visualstudio.microsoft.com/downloads/) page and get the "Build Tools for Visual Studio 2022". After the download completes, run the installer to get Microsoft Visual C++.

---

If the `python` command in **Git Bash** freezes and never gives you a `>>>` prompt, then we need to use a program called [winpty](https://stackoverflow.com/questions/48199794/winpty-and-git-bash) to run the Python executable. This is the command for running Python with winpty:

```bash
(.venv) $ winpty python.exe
```

We can create an [alias](https://en.wikipedia.org/wiki/Alias_(command)) in our `.bashrc` file so that the `python` command always runs with winpty. The `.bashrc` script runs every time we open the Git Bash, so it is a good place to put things like aliases that you want to use every time. Type this command to add the alias to your `.bashrc` file:

```bash
(.venv) $ echo "alias python='winpty python.exe'" >> ~/.bashrc
```

Here, the `alias` command assigns the `winpty python.exe` command to `python`. Here we use `echo` and `>>` to append the alias command to the end of the `.bashrc` file in the user's home directory. We can confirm the command is set by looking at the contents in the `~/.bashrc` file.

```bash
(.venv) $ tail ~/.bashrc
alias python='winpty python.exe'
```

Finally, restart the bash or type `source ~/.bashrc` to run the script.

## Troubleshooting on MacOS

When testing your OpenGL installation, the `import OpenGL.GL` statement may print a long traceback and an `ImportError` that says something like `'Unable to load OpenGL library'`. If that happens, we need to manually change some of the OpenGL source code in the Python library. Find and open the `ctypesloader.py` file inside the `OpenGL/platform` directory. If you followed the instructions for creating your virtual environment above, the path from your project directory should be:  

`.venv/lib/python3.7/site-packages/OpenGL/platform/`

Inside the `ctypesloader.py` file, find the line that has:

```python
fullName = util.find_library( name )
```

and change it to:

```python
fullName = f"/System/Library/Frameworks/{name}.framework/{name}"
```

:warning:**&nbsp;Be careful you do not change the original indentation!&nbsp;**:warning:

Save the file and then test `import OpenGL.GL` in the interactive Python interpreter once more:

```bash
(.venv) $ python
Python 3.7.3 (v3.7.3:ef4ec6ed12)
Type "help", "copyright", "credits" or "license" for more information.
>>> import OpenGL.GL
>>> exit()
(.venv) $ █
```

If you don't get an error, then you are ready to start the project!