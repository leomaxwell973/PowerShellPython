# PowerShellPython

A modified `subprocess.py` (`subprocess.run`) that prefixes code to replace core shell operations from `cmd.exe` with `PowerShell`, targeting commands like **ninja, cmake, cl/msvc, link, etc.** The goal is to **bypass cmd.exe issues** by using PowerShell for Python execution.

This Project has 3 Goals:
1. Bypass CMD issues causing build/install fails.
2. Bring seamless pip ecosystem back to Windows.
3. Make Windows more viable and functionality less dependent on Linux VM-ware or other invasive workarounds.

## Features

- **Portable & Future-Proof** – No additional imports; designed for maximum compatibility with minimal invasiveness.
- \*Modification Limited to\* **`subprocess.run`** – The rest of `subprocess.py` remains untouched.
- **Works with Standard Build Toolchains** – Requires the usual dependencies: MSVC, C++, CL, VC (Visual Studio), CMake (Windows version, *not Python’s*), Ninja (Windows version, not Python’s), CUDA, etc.
- **Fixes vcvarsall.bat Conflicts** – Uses a **custom call** to initialize MSVC without interference from PowerShell prefixing.
- **Supports Different PowerShell Versions** – While tested on **PowerShell v1.0**, it should work with other versions based on pathing/env priority.
- **Validated on Both Native & VM Environments** – Tested with **Python 3.10.6, CUDA 12.1, PyTorch 2.5.1+cu121, Xformers v0.0.29.post1, and Flash-Attn v2.7.4.post1**.
- \*Works with Plain\* **`pip install`** – No need for manual builds or Git repo installations.

## Recommended Fixes & Enhancements

### **Optional: PowerShell Helper Script (ver.ps1)**

This **optional** script improves **verbose logging compatibility**. Place it in `system32` or another accessible location and **enable execution** via:

```powershell
Set-ExecutionPolicy Unrestricted
```

**ver.ps1:**

```powershell
$os = Get-CimInstance -ClassName Win32_OperatingSystem
Write-Output ""
Write-Output "Microsoft Windows [Version $($os.Version)]"
```

### Highly Recommended: Setuptools Fix (`build.py` Patch)

Issue in `build.py` involving `+ plat_specifier` can cause Windows pip install failures (e.g., Xformers). Remove the lines to resolve it:

**`setuptools/_distutils/command/build.py`**:

```python
if self.build_purelib is None:
    self.build_purelib = os.path.join(self.build_base, 'lib')  # <<< Remove + plat_specifier
if self.build_platlib is None:
    self.build_platlib = os.path.join(self.build_base, 'lib')  # <<< Remove + plat_specifier
...
self.build_temp = os.path.join(self.build_base, 'temp')  # <<< Remove + plat_specifier
```

A **scripted auto-fix** is included in PowerShellPython but is **commented out by default**.

---

## **PowerShellPython - Main Code (**`subprocess.py` Modifications)

This code is **inserted inside** `subprocess.py` after `def run` but before the rest of `subprocess.run`.

```python
def run(*popenargs,
        input=None, capture_output=False, timeout=None, check=False, **kwargs):
    """
    Run command with arguments and return a CompletedProcess instance.
    (Docstring omitted for brevity)
    
    PowerShellPython - Intercept and modify command arguments
    """
    # Uncomment for debugging
    # print("---RUN ATTEMPT EXECUTED (UN-Modified):", popenargs, kwargs)
    
    # Ensure MSVC environment is initialized only if DISTUTILS_USE_SDK is NOT set
    if os.environ.get("DISTUTILS_USE_SDK") != "1":
        try:
            vcvarsall_path = r"C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvarsall.bat"
            print(f"Running MSVC initialization: {vcvarsall_path} x64")

            # Run vcvarsall.bat and capture the output environment variables
            with os.popen(f'cmd.exe /c "set DISTUTILS_USE_SDK=1 && call \"{vcvarsall_path}\" x64 && set"') as pipe:
                output = pipe.read()

            # Parse the environment variables and set them in Python
            for line in output.splitlines():
                if '=' in line:
                    key, value = line.split('=', 1)
                    os.environ[key] = value.strip()

            # Confirm the variable was set
            if os.environ.get("DISTUTILS_USE_SDK") == "1":
                print("MSVC environment successfully initialized.")
            else:
                raise RuntimeError("MSVC initialization failed: DISTUTILS_USE_SDK not set.")

        except Exception:
            vcvarsall_path = r"C:\Program Files (x86)\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvarsall.bat"
            print(f"Primary MSVC initialization failed. Trying alternative: {vcvarsall_path} x64")

            try:
                with os.popen(f'cmd.exe /c "set DISTUTILS_USE_SDK=1 && call \"{vcvarsall_path}\" x64 && set"') as pipe:
                    output = pipe.read()

                # Apply environment variables
                for line in output.splitlines():
                    if '=' in line:
                        key, value = line.split('=', 1)
                        os.environ[key] = value.strip()

                if os.environ.get("DISTUTILS_USE_SDK") == "1":
                    print("MSVC environment successfully initialized.")
                else:
                    raise RuntimeError("MSVC initialization failed: DISTUTILS_USE_SDK not set.")

            except Exception as e:
                print(f"ERROR: Failed to initialize MSVC environment. Compilation may fail! ({e})")

    if isinstance(popenargs[0], str):
        cmd_list = popenargs[0].replace("&&", "AND")
        
    else:
        cmd_list = list(popenargs[0])
        cmd_list = [arg.replace("&&", "AND") for arg in cmd_list]
        if popenargs[0] == "ninja":
            script_dir = os.path.dirname(os.path.abspath(__file__))  # Get the directory of the script
            file_path = os.path.join(script_dir, "site-packages/setuptools/_distutils/command/build.py")
            
            if os.path.exists(file_path):
                
                with open(file_path, 'r') as f:
                    for line_num, line in enumerate(f, start=1):
                        if " + plat_specifier" in line:
                            print(f"WARNING: PowerShellPython found ' + plat_specifier' in setuptools/_distutils/command/build.py at line {line_num}.")
                            print(f"this may cause some pip install-builds to fail on windows! (I.E. Xformers)")
                            
               ## if okay to overwrite automatically, use below (workaround for automatic updates etc.)
               ## and comment/remove the above warning block.
               # with open(file_path, 'r') as f:
               #     lines = f.readlines()
               # with open(file_path, 'w') as f:
               #     f.writelines(line.replace(" + plat_specifier", "") for line in lines)
                    
            else:
                print(f"WARNING: PowerShellPython did not find setuptools!")

    if popenargs[0] in ["cmd", "cmd.exe"]:
        cmd_list[0] = cmd_list[0].replace("cmd", "powershell")
        cmd_list[1] = cmd_list[1].replace("/c", "-Command")
    popenargs = (cmd_list,)
    
    try:
        # Run `where powershell` safely without subprocess.run()
        with os.popen("where.exe powershell") as pipe:
            powershell_path = pipe.read().strip()

        # Extract the first valid path
        powershell_path = powershell_path.split("\n")
        powershell_path = powershell_path[0].strip()  # Return the first valid path
        
    except Exception:
        powershell_path = "C:/Windows/System32/WindowsPowerShell/v1.0/powershell.exe"  
        # If `where` fails, fall back to default path    
    if kwargs.get("shell") is True:
        kwargs["executable"] = powershell_path
    
    # Uncomment for debugging
    # print("---RUN EXECUTED (Modified):", popenargs, kwargs)
```

---

## **Installation & Usage**

1. **Backup your current** `subprocess.py` 
2. **Replace** `subprocess.py` with the **modified version** 
   1. (**Optional**) Arrange original copy of subprocess.py and the powershellpython version so they can be swapped or toggled.
      I.E.
      subprocess.py / subprocess_original.py (when running powershellpython)
      &
      subprocess.py / subprocess_powershellpython.py (when running original subprocess)
3. Apply the `build.py` patch if needed
4. (Optional) Add `ver.ps1` to improve log handling

---

### **Conclusion**

PowerShellPython aims to make **Windows AI development smoother** by replacing `cmd.exe` with **PowerShell** where possible. While not a universal fix, it has shown **significant improvements in AI package installations and builds**, reducing cmd-related failures.

This is a work-in-progress—feedback is welcome!

