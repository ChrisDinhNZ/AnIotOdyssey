---
layout: post
title: Managing Python Environments and Dependencies
date: 2021-06-06
background: '/assets/posts/2021-06-06-managing-python-environments-and-dependencies/post-banner-2021-06-06-managing-python-environments-and-dependencies.jpg'
Tag:
    - Python
    - Tooling
---

# Managing Python Environments and Dependencies

The great thing about Python is that whatever problems (including IoT problems) you are trying to solve, there is a very high chance there’s is a Python library out there that’s going to make your life a lot easier. The troublesome thing about Python is that, even at this point in time, there is still two Python universe and it makes life very hard for those that wants to move forward.

Let’s consider the following scenario:

* You have a solution that depends on a set of libraries and at specific versions.
* Those libraries also depends on other libraries and also at specific versions.
* Runs on Python 2.7
* Libraries owners may not have the time to migrate to Python 3.

If you take the above scenario and try to migrate the solution to Python 3, it can be a daunting task.

## Managing Python environments in Windows

The purpose of this post is not solving our migration problems but more about keeping things nice and tidy, manageable, and us developers happy. To do that I will discuss some concepts that every Python developers should be familiar with:

* Python Launcher
* Virtual Environments
* Pip

**Python Launcher**

Python Launcher is a Windows utility used to identify which Python versions as well as where they are installed. It also provide a mechanism to launch Python programs with a specified Python interpreter.

For my development environment, I know that I need to work with both Python 2 and Python 3 so let's head over to [python.org download page](https://www.python.org/downloads/) download the latest version for each (as of this writing they were [Python 3.9.5](https://www.python.org/downloads/release/python-395/) and [Python 2.7.18](https://www.python.org/downloads/release/python-2718/)). Also since Python Launcher was introduced in Python 3.3, it will be included in `Python 3.9.5`.


![Installing Python 2.7.18](/assets/posts/2021-06-06-managing-python-environments-and-dependencies/python_2_7_installation.gif)
_Installing Python 2.7.18_

![Installing Python 3.9.5](/assets/posts/2021-06-06-managing-python-environments-and-dependencies/python_3_9_installation.gif)
_Installing Python 3.9.5_

After installation has been completed, open a terminal window and execute the command “py –help” which will print out the help for “py” (Python Launcher).

![Python launcher command demo](/assets/posts/2021-06-06-managing-python-environments-and-dependencies/py_command_demo.gif)
_Python launcher command demo_

**Virtual Environments**

Python virtual environment allows developers to create self-contained environments when working in Python. My preferred workflow is to create a virtual environment per project I work with so that Python version and external libraries are only relevant to the environment they were created in. Below are examples of create a Python 2 and Python 3 virtual environment, as well as activating and deactivating them.

```
    PS D:\Workspace> mkdir python2-project-demo
        Directory: D:\Workspace
    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    d----           5/06/2021 08:45 pm                python2-project-demo
    PS D:\Workspace> cd .\python2-project-demo\
    PS D:\Workspace\python2-project-demo> py -2 -m virtualenv venv
    created virtual environment CPython2.7.18.final.0-64 in 8526ms
    creator CPython2Windows(dest=D:\Workspace\python2-project-demo\venv, clear=False, no_vcs_ignore=False, global=False)
    seeder FromAppData(download=False, pip=bundle, wheel=bundle, setuptools=bundle, via=copy, app_data_dir=C:\Users\Chris\AppData\Local\pypa\virtualenv)
        added seed packages: pip==20.3.4, setuptools==44.1.1, wheel==0.36.2
    activators PythonActivator,FishActivator,BatchActivator,BashActivator,PowerShellActivator
    PS D:\Workspace\python2-project-demo> ls
        Directory: D:\Workspace\python2-project-demo
    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    d----           5/06/2021 08:46 pm                venv
    PS D:\Workspace\python2-project-demo> .\venv\Scripts\activate
    (venv) PS D:\Workspace\python2-project-demo> python -V
    Python 2.7.18
    (venv) PS D:\Workspace\python2-project-demo> deactivate
    PS D:\Workspace\python2-project-demo>
```

```
    PS D:\Workspace> mkdir python3-project-demo
        Directory: D:\Workspace
    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    d----           6/06/2021 08:50 pm                python3-project-demo
    PS D:\Workspace> cd .\python3-project-demo\
    PS D:\Workspace\python3-project-demo> py -3 -m venv venv
    PS D:\Workspace\python3-project-demo> ls
        Directory: D:\Workspace\python3-project-demo
    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    d----           6/06/2021 08:51 pm                venv
    PS D:\Workspace\python3-project-demo> .\venv\Scripts\activate
    (venv) PS D:\Workspace\python3-project-demo> python -V
    Python 3.9.5
    (venv) PS D:\Workspace\python3-project-demo> deactivate
    PS D:\Workspace\python3-project-demo>
```

**Pip**

Pip is a Python package installer. It is used for installing and managing external libraries for a given Python environment.

One reason I use Python is when working with CoAP. CoAP is a client/server transport protocol ideally used for communicating with constraint resource devices. In Python 2.7, CoAPthon is a good library to use. While in Python 3, Aiocoap is a nice option.

Using the Python 3 virtual environment created earlier, let's take a look at some of the useful pip commands.

For installing external packages, you can use `pip install <package-to-install>`.

```
    (venv) PS D:\Workspace\python3-program> pip install aiocoap
    Collecting aiocoap
    Using cached aiocoap-0.4.1.tar.gz (201 kB)
    Building wheels for collected packages: aiocoap
    Building wheel for aiocoap (setup.py) ... done
    Created wheel for aiocoap: filename=aiocoap-0.4.1-py3-none-any.whl size=181210 sha256=09fafe85ee179a31e52bb57b088af1fcdf3e4114b664c6af7615af5715feb54c
    Stored in directory: c:\users\chris\appdata\local\pip\cache\wheels\2c\a6\f8\63466bc1a04e8fb6e88707a6005a65529e451f02221e88f542
    Successfully built aiocoap
    Installing collected packages: aiocoap
    Successfully installed aiocoap-0.4.1
    (venv) PS D:\Workspace\python3-program> 
```

You can use `pip list` to list all installed packages.

```
    (venv) PS D:\Workspace\python3-program> pip list
    Package    Version
    ---------- -------
    aiocoap    0.4.1
    pip        21.1.2
    setuptools 56.0.0
    wheel      0.36.2
    (venv) PS D:\Workspace\python3-program>
```

You can use `pip show <package-to-show>` to show the specified package's details.

```
    (venv) PS D:\Workspace\python3-program> pip show aiocoap
    Name: aiocoap
    Version: 0.4.1
    Summary: Python CoAP library
    Home-page: https://github.com/chrysn/aiocoap
    Author: Maciej Wasilak, Christian Amsüss
    Author-email: c.amsuess@energyharvesting.at
    License: MIT
    Location: d:\workspace\python3-program\venv\lib\site-packages
    Requires:
    Required-by:
    (venv) PS D:\Workspace\python3-program>
```

You can use “pip list –format=freeze > requirements.txt” to export the list to a file which could then be used in another virtual environment.

```
    (venv) PS D:\Workspace\python3-program> pip list --format=freeze > requirements.txt
    (venv) PS D:\Workspace\python3-program> Get-Content .\requirements.txt
    aiocoap==0.4.1
    pip==21.1.2
    setuptools==56.0.0
    wheel==0.36.2
    (venv) PS D:\Workspace\python3-program> deactivate
    PS D:\Workspace\python3-program> cd ..
    PS D:\Workspace> mkdir python3-program-v2
    
        Directory: D:\Workspace
    
    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    d----           7/06/2021 8:46 pm                python3-program-v2
    
    PS D:\Workspace> cd .\python3-program-v2\
    PS D:\Workspace\python3-program-v2> py -3 -m venv venv
    PS D:\Workspace\python3-program-v2> .\venv\Scripts\activate
    (venv) PS D:\Workspace\python3-program-v2> pip list
    Package    Version
    ---------- -------
    pip        21.1.1
    setuptools 56.0.0
    WARNING: You are using pip version 21.1.1; however, version 21.1.2 is available.
    You should consider upgrading via the 'd:\workspace\python3-program-v2\venv\scripts\python.exe -m pip install --upgrade pip' command.
    (venv) PS D:\Workspace\python3-program-v2> python.exe -m pip install --upgrade pip
    Requirement already satisfied: pip in d:\workspace\python3-program-v2\venv\lib\site-packages (21.1.1)
    Collecting pip
    Using cached pip-21.1.2-py3-none-any.whl (1.5 MB)
    Installing collected packages: pip
    Attempting uninstall: pip
        Found existing installation: pip 21.1.1
        Uninstalling pip-21.1.1:
        Successfully uninstalled pip-21.1.1
    Successfully installed pip-21.1.2
    (venv) PS D:\Workspace\python3-program-v2> cp ..\python3-program\requirements.txt .
    (venv) PS D:\Workspace\python3-program-v2> pip install -r .\requirements.txt
    Collecting aiocoap==0.4.1
    Using cached aiocoap-0.4.1-py3-none-any.whl
    Requirement already satisfied: pip==21.1.2 in d:\workspace\python3-program-v2\venv\lib\site-packages (from -r .\requirements.txt (line 2)) (21.1.2)
    Requirement already satisfied: setuptools==56.0.0 in d:\workspace\python3-program-v2\venv\lib\site-packages (from -r .\requirements.txt (line 3)) (56.0.0)
    Collecting wheel==0.36.2
    Using cached wheel-0.36.2-py2.py3-none-any.whl (35 kB)
    Installing collected packages: wheel, aiocoap
    Successfully installed aiocoap-0.4.1 wheel-0.36.2
    (venv) PS D:\Workspace\python3-program-v2> pip list
    Package    Version
    ---------- -------
    aiocoap    0.4.1
    pip        21.1.2
    setuptools 56.0.0
    wheel      0.36.2
    (venv) PS D:\Workspace\python3-program-v2>
```

## Installing the latest Python interpreter on Linux (Raspberry Pi OS)

As of writing, Python 3.9 was the latest version, however this was not installed by default.

![Check installed Python demo](/assets/posts/2021-06-06-managing-python-environments-and-dependencies/python3.9_not_found_demo.gif)
_Check installed Python demo_

You can install Python 3.9 from source. [Python 3.9 install on Raspberry PI OS](https://community.home-assistant.io/t/python-3-9-install-on-raspberry-pi-os/241558) is a good post showing how you can go about doing that.

![Install from source demo](/assets/posts/2021-06-06-managing-python-environments-and-dependencies/python3.9_install_from_source_demo.gif)
_Install from source demo_