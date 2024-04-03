---
# MARP
theme: default
paginate: true

# Jekyll
title: "1_Getting Started"
date: 2024-04-01
categories:
  - Guides
---

*OpenGL programming in Python requires a little effort to set up the development environment. This post tells you how.*

# Software Requirements

In the lessons that follow, We will build a framework for interactive computer graphics applications written in Python with the OpenGL library. For this project, we need to have the following software installed:

- Python 3.7 or higher
- PyGame
- Numpy
- PyOpenGL
- PyOpenGL-accelerate

This guide will show you how to set up and confirm your development environment now so you do not have any surprises later during the project.

I recommend installing everything in a virtual environment which can be turned on and off as needed. Then the software configurations for this project will not interfere with other projects you may have on the same computer.

This guide will explain everything about setting up and using a virtual environment for the required packages.

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
brew install python3
```

Once the installation completes, you can confirm the version with `python3 --version` again.

## Windows

On **Windows**, you can upgrade Python using an installer from the [Python Downloads](https://www.python.org/downloads/windows/) page. Just open the page and click the "Latest Python 3 Release" link.

If you get the option during installation, be sure add Python to your `PATH` variable. 

After the installation completes, open a new terminal and confirm the version number with `python --version` again. 

# Virtual Environment

Once we have Python 3 installed, we can create a virtual environment using its built-in `venv` module. According to the [docs](https://docs.python.org/3/library/venv.html), `venv` is a simple virtual environment tool which provides the following benefit:

> Each virtual environment has its own Python binary (which matches the version of the binary that was used to create this environment) and can have its own independent set of installed Python packages in its site directories.

This means we can create an isolated installation of Python with only the necessary packages and we don't need to worry about breaking other projects or software on our PC.

First, make a directory for this project somewhere on your computer and navigate to it in your terminal. Then type the following command:
 
```bash
$ python3 -m venv .venv
```

This will use the `venv` module to create a virtual environment with your Python 3 binaries in a folder called `.venv`. I like to name the environment folder with a dot (`.`) so it is easy to distinguish and always appears at the top of directory listings. This also hides the folder by default when viewing files in programs like Finder on MacOS or File Explorer on Windows.

Before we can start using the virtual environment, we must activate it. Fortunately, the environment we just created provides a binary script in its `bin` folder just for that purpose. Run the script with the following command in the same place where you created the `.venv` folder:

**MacOS**
```bash
source .venv/bin/activate
```

**Windows**
```bash
source .venv/Scripts/activate
```

:warning:**&nbsp;Notice the space and dot `.` after `source`&nbsp;**:warning:

Now you should see the name of your virtual environment in parentheses at the front of your command prompt. For example, my prompt changes from `$` to `(.venv) $ `. This means you are working inside the active virtual environment and any Python modules you install will be exclusive to that environment. If you try the `pip list` command now, you will probably see that only the bare minimum packages are installed:

```bash
(.venv) $ pip list
Package    Version
---------- -------
pip        20.2.3
setuptools 49.2.1
```
:arrow_right: Your terminal will likely show different version numbers for the packages above. That's okay. We will upgrade pip in the next section.

With `.venv` we have a clean, minimal environment in which we can freshly install only the things we need.

Whenever you finish working in the environment, always remember to deactivate it so you don't accidentally use it on another project. You can do this simply by typing `deactivate` in the terminal.

```bash
deactivate
```

:warning:**&nbsp;Always `activate` before you start and `deactivate` when you stop working.&nbsp;**:warning:

# Python Packages

All the required software packages can be installed with `pip` which automatically finds the necessary source, downloads them, and installs them. However, `pip` needs some manual configuration of its network settings if you are using it behind a proxy server, which is the case on the KIT school network.

## `pip` Proxy Configuration

The `pip list -o` command gets the latest versions of your installed packages from package repositories. We can use this command to test whether `pip` can connect or not. Try typing `pip list -o` into your terminal. If it gives an error, then you might need to configure proxy settings for `pip` to work correctly.

Here we will configure `pip` to use the KIT internal network proxy server only when our virtual environment is active.

First, activate the virtual environment as mentioned above and then type the following command:

```bash
pip config --site set global.proxy http://wwwproxy.kanazawa-it.ac.jp:8080
```

You should see a message that says it is writing to a `pip.conf` file (for **MacOS**) or a `pip.ini` file (for **Windows**). Make sure the file is inside your virtual environment `.venv` directory.

If this doesn't work, you can create a blank `pip.conf` or `pip.ini` file inside your `.venv` directory and save it with these contents:
```
[global]
proxy = http://wwwproxy.kanazawa-it.ac.jp:8080
```

Finally, confirm that the proxy is set with the following command in your terminal and you should see the proxy server address echoed back to you.

```bash
(.venv) $ pip config get global.proxy
http://wwwproxy.kanazawa-it.ac.jp:8080
```

Now confirm that `pip` can communicate through the proxy with the `pip list -o` command again. If you do not get an error but see a list of package names and version numbers instead, then you are ready to install packages!

By the way, when you are not behind a proxy (such as off the campus network) and want to use `pip` you should disable the proxy setting. The `unset` command can be used for this purpose:

```bash
pip config --site unset global.proxy
```

Of course, deactivating the virtual environment will also disable the proxy setting for `pip`. But be careful because any packages you install after deactivating will apply to your default Python installation.

## Installing Packages

It is always good to use the latest version of `pip` to make sure everything is up-to-date, so let's upgrade it first. With your `.venv` environment active, run the command:

```bash
python -m pip install --upgrade pip
```

:fire: Notice we are not using the `python3` command anymore. We created the `.venv` virtual environment with Python 3, so the `python` command uses Python 3 automatically while the environment is active. :sunglasses:

You will see a message that says `Successfully installed pip-XX.XX` when the upgrade completes.

Now we can install the required packages:
```bash
pip install pygame numpy pyopengl pyopengl-accelerate
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

If that happens, open the [Visual Studio Downloads](https://visualstudio.microsoft.com/downloads/) page and search "All Downloads" for "Build Tools for Visual Studio 2022". Click the "Download" button and run the installer to get the necessary Microsoft Visual C++ tools.

When the installation completes, try installing PyOpenGL and PyOpenGL-accelerate again to confirm that the problem is resolved.  

---

If the `python` command in **Git Bash** freezes and never gives you a `>>>` prompt, then we need to use a program called [winpty](https://stackoverflow.com/questions/48199794/winpty-and-git-bash) to run the Python executable. This is the command for running Python with winpty:  

```bash
winpty python.exe
```  

But typing that out everytime can be troublesome. Instead, we can create an [alias](https://en.wikipedia.org/wiki/Alias_(command)) for the `python` command and save it to our `.bashrc` script. The `.bashrc` script runs every time we open the Git Bash, so it is a good place to put things like aliases that you want to use in every session. Type this command to add the alias to your `.bashrc` file:  

```bash
echo "alias python='winpty python.exe'" >> ~/.bashrc
```  

Here, the `alias` command assigns the `winpty python.exe` command to the `python` alias. We use `echo` and `>>` to append the alias command to the end of the `.bashrc` file in the user's home directory. Confirm the command is set by looking at the last few lines in the `~/.bashrc` file with the `tail` command:  

```bash
(.venv) $ tail ~/.bashrc
alias python='winpty python.exe'
```  

Finally, run the script by either reopening the bash window or typing `source ~/.bashrc`. Now simply typing `python` in the terminal will run the `winpty python.exe` command automatically!  

## Troubleshooting on MacOS  

When testing your OpenGL installation, the `import OpenGL.GL` statement may print a long traceback with an `ImportError` that says something like `'Unable to load OpenGL library'`. If that happens, we need to manually change some of the OpenGL source code in the Python library. Find the `OpenGL/platform` directory within your virtual environment's Python library and open the `ctypesloader.py` file. If you followed the instructions for creating your virtual environment above, the path from your project directory should be:  

`.venv/lib/python3.7/site-packages/OpenGL/platform/`  

Inside the `ctypesloader.py` file, find the line that has:  

```python
fullName = util.find_library( name )
```  

and change it to:  

```python
fullName = f"/System/Library/Frameworks/{name}.framework/{name}"
```  

:warning:**&nbsp;Be careful not to change the original indentation!&nbsp;**:warning:

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
