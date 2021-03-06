# PyPI publishing instructions
Mostly following official instructions from [https://packaging.python.org/tutorials/packaging-projects/] (https://packaging.python.org/tutorials/packaging-projects/), with <strike>minor</strike> comments and modifications

## Step 0 (Ideas)

`pip install` uses two options to install pacakges. 
1. Installation from source, where the packages are built and compiled from source. The whole project is zipped to a `tar.gz`, extracted and compiled on the target computer. 
2. Installation from `.wheel` (zip file of comiled code). If the wheel for the target system and python version is available, wheel is downloaded, unzipped and library is ready to use. This is the preferred way of doing it for the end user, but you must compile it for each system seperately.

## Step 1 (Prepare pacakge)
Add `setup.py` file for installation entry point and `MANIFEST.in` file that adds additional files. 

`setup.py` file handles the whole instalation procedure. The whole point of `my_build_ext` in `setup.py` is to get around the fact that `numpy` bindings are needed during the compilation, but `numpy` is not yet available at the start of instalation. `FLAGS` are directly copied from `MAKEFILE` and `numpy.get_include()` is needed to correctly link to its objects (flags need to be updated to correctly handle microsoft visual cpp compiler). `cpp_sources` automatically includes all `.cpp` files present in the `code`. Take care, that included files are specified with a relative path, otherwise package is invalid (but you won't be notified automatically). The files are to be compiled as part of the instalation. `swig_lib` defines the extension to be compiled the name argument is important, as the resulting file is compiled as `_streamdm.so` (file extension is platform dependent) in our case. Setup part of the instalation is just info presented on pypi webpage.  

`Manifest.in` is required to include additional `.h` files that are needed in the compilation. Those files are copied to instalation zip  and distributed. Files referenced in `setup.py` are automatically included and do not need to be defined by `Manifest.in`, but including them is not an error.

## Step 2 (Prepare code)

Add additional files to conclude module.
Files `__init__.py` (to ensure that module is detected by Python), `test.py` (to ship tests) and `streamdm.py`, which is generated by `SWIG`. 

**!!!IMPORTANT!!!**
`streamdm.py` is almost the swig generated file except for a minor change. Due to importing style (in a module), swig imports compiled file as 
```python
# Import the low-level C/C++ module
if __package__ or "." in __name__:
	from . import _streamdm
else:
	import _streamdm
```
Which results in error, as it tries to reimports itself (the entry point for module import is `__init__.py` file, which should import everything) which results in cyclic import.

Generated code must therefore be replaced just with the `else` branch:
```python
# Import the low-level C/C++ module
import _streamdm
```
This must be done manually after every invocation of swig.
Just for information, installed directory structure looks like this
```
venv
|- ...
|- lib
   |- python3.8 (version_number)
	  |- site-pacakges
		 |- ... (all installed packages
		 |- ml_rapids
		    | __init__.py
		    | streamdm.py
		    | test.py (directly coped from ml_rapids folder)
		 | _streamdm.cpython-38-darwin.so (compiled file: version_number and platform)
		 | easy_install.py
```

Swig must deliberately be invoked on machine praparing the instalation, as this enables users to install library from sources without swig. To see how to do it, see `makefile` and development instructions.

## Step 3: Package 

After steps 1 and 2, you are ready to publish. Follow the official instructions if anything is unclear. First set up account on [https://pypi.org/](https://pypi.org/) and generate the key. To make things easier, you might consider seting up `.pypirc` file, so as not to copy and pase it every time. (I had some troubles gatting this ready on windows.)

Install packages needed to build and publish: `pip install setuptools wheel twine` (twine is optional, but eases the publishing processs, wheel is needed to be able to build wheels).

Build package with: 
`python setup.py sdist bdist_wheel`  
This builds pacakge and creates folder `dist` where built pacakges are kept. Two commands are executed: 
* `sdist` Which builds source distribution (just copies all the files and zips them in `dist/ml-rapids-version_number.tar.gz`. This does not compile anything, but the pacage can already be published
* `bdist_wheel`: Which builds binary wheel distribution. `ml_rapids-0.0.1.4-cp38-cp38-macosx_10_9_x86_64.whl` (ml_rapids-version-python-version-system.whl) This is the wheel, that can be reused on the same system and python version. This command compiles everything and takes some time (you also get all the compiler warnings and so). It is functionally the same as `pip isnstall ml-rapids`, where the computer just downloads source code and builds it. The process seems smart, as rebuilds are only triggered on file changes (so if you compile multiple times, it caches builds).

The commands can be run seperately to rebuild just a part of distributions (just `bdist_wheel` or `sdist`).


## Step 4 (Publish)

You just need to upload everything to pypi.
If you made a change, do not forget, to increase version number in `setup.py`. 

`twine upload dist/*` will uplad all files in dist folder to pypi. You will be prompted for password if you haven't set up the `.pypirc` file (follow: [https://packaging.python.org/tutorials/packaging-projects/#uploading-the-distribution-archives](https://packaging.python.org/tutorials/packaging-projects/#uploading-the-distribution-archives)). The files are uploaded to pypi. 

You may build packages for multiple python versions simply by creating virtualenvironments on same system, running `setup.py` under them and upload multpile pacakges together (source does not change, just wheels are different).


**!!BEWARE!!** 
When uploading new versions, older are made redundant. If you compiled a whole bunch of wheels, but then increase the version, `pip install` will try to install the latest version, if no wheels are available (since existing wheels are now old, no wheels are available), package will be build from source on target computers. 


# Building a new release automagically with [cibuildwheel](https://github.com/joerick/cibuildwheel) and Travis

## Prepare Travis
- Enable project in [Travis](https://travis-ci.org/dashboard) (you have to be admin/have sufficient permissions)
- Add pypi token to environment variable `TWINE_PASSWORD` to Travis config (on Travis webpage). This enables automatical upload of build wheels.



## Release new version

Travis is configured to build any push on `release` branch automatically and push built wheels and source distribution to pypi. Already existing versions of wheels are ignored and not uploaded.

Release checklist (Part 2 and 3 should be automated soon) (Running `make swigfiles` should generate all required files):
1. Make sure you have an up to date repository state
1. Generate fresh `.cxx` file and push it to git: 
    - Generate fresh `streamdm_wrap.cxx` and commit it to repository.
1. Generate fresh `ml_rapids/streamdm.py` and fix it manually:
    - Replace
        ```python
        # Import the low-level C/C++ module
        if __package__ or "." in __name__:
            from . import _streamdm
        else:
            import _streamdm
        ```
        with
        ```python
        # Import the low-level C/C++ module
        import _streamdm
        ```
        somewhere around line 10.
    - Commit `ml_rapids/streamdm.py`
1. Increase version in `setup.py` (If you don't increase the version, new wheels won't get uploaded, as they already exist)
1. Merge changes to `release` branch on main repository and push (or open pull request and confirm it).
1. Wait for travis to build the wheels and source distribution (Full process takes up to an hour, first wheels get published a lot faster).
