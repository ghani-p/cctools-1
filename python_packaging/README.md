# Python Packaging Utilities

The Python packaging utilities allow users to easily analyze their Python scripts and create Conda environments that are specifically built to contain the necessary dependencies required for their application to run. In distributed computing systems such as Work Queue, it is often difficult to maintain homogenous work environments for their Python applications, as the scripts utilize a large number of outside resources at runtime, such as Python interpreters and imported libraries. The Python packaging collection provides three easy-to-use tools that solve this problem, helping users to analyze their Python programs and build the appropriate Conda environments that ensure consistent runtimes within the Work Queue system. 

The `python_package_analyze` tool analyzes a Python script to determine all its top-level module dependencies and the interpreter version it uses. It then generates a concise, human-readable JSON output file containing the necessary information required to build a self-contained Conda virtual environment for the Python script.

The `python_package_create` tool takes the output JSON file generated by `python_package_analyze` and creates this Conda environment, preinstalled with all the necessary libraries and the correct Python interpreter version. It then generates a packaged tarball of the environment that can be easily relocated to a different machine within the system to run the Python task.

The `python_package_run` tool acts as a wrapper script for the Python task, unpacking and activating the Conda environment and running the task within the environment.






# python_package_analyze(1)

## NAME

`python_package_analyze` - command-line utility for analyzing Python script for library and interpreter dependencies

## SYNOPSIS

`python_package_analyze [options] <python-script> <json-output-file>`

## DESCRIPTION

`python_package_analyze` is a simple command line utility for analyzing Python scripts for the necessary external dependencies. It generates an output file that can be used with `python_package_create` to build a self-contained Conda environment for the Python application.

The `python-script` argument is the path (relative or absolute) to the Python script to be analyzed. The `json-output-file` argument is the path (relative or absolute) to the output JSON file that will be generated by the command. The file does not need to exist, and will overwrite a file with the same name if it already exists.

## OPTIONS

-h        Show this help message

## EXIT STATUS

On success, returns zero. On failure, returns non-zero.
- 1 - Invalid command format
- 2 - Invalid path to the Python script to be analyzed

## EXAMPLE

An example Python script `example.py` contains the following code:

```
import os
import sys
import pickle

import antigravity
import matplotlib


if __name__ == "__main__":
    print("example")
```

To analyze the `example.py` script for its dependencies and generate the output JSON dependencies file `dependencies.json`, run the following command:

`$ python_package_analyze example.py dependencies.json`

Once the command completes, the `dependencies.json` file within the current working directory will contain the following, when the default `python3` interpreter on the local machine is Python 3.7.3:

`{"python": "3.7.3", "modules": ["antigravity", "matplotlib"]}`

Note that system-level modules are not included within the `"modules"` list, as they are automatically installed into Conda virtual environments. Additionally, using a different version of the Python interpreter will result in a different mapping for the `"python"` value within the output file.

## POSSIBLE IMPROVEMENTS
1. Utilize `ModuleFinder` library to get complete list of modules that are used by the Python script
- Provides more comprehensive list of modules used, including system-level modules, making it redundant
- Takes longer to run compared to the currently-implemented parsing algorithm
- More rigorously tested than the parsing algorithm, so it ensures that all modules will be listed
2. Use `pip freeze` to find all modules that are installed within the machine
- Instead of seeing if the module is not a system module, just see if it is installed on the machine, but requires that the module be installed on the master machine
- Misses cases where a module is installed to the machine, but not by pip
- The advantage to this option is that `pip freeze` includes versions, so you can add version numbers for module dependencies to get more accurate pip installations into the virtual environment
- `stdlib_list` library that is in the current implementation requires installation and has not been rigorously tested



# python_package_create(1)

## NAME

`python_package_create` - command-line utility for creating a Conda virtual environment given a Python dependencies file

## SYNOPSIS

`python_package_create [options] <dependency-file> <environment-name>`

## DESCRIPTION

`python_package_create` is a simple command-line utility that creates a local Conda environment from an input JSON dependency file, generated by `python_package_analyze`. The environment is installed into the default Conda directory on the local machine. The command also creates an environment tarball in the current working directory with extension `.tar.gz` that can be sent to and run on different machines with the same architecture.

The `dependency-file` argument is the path (relative or absolute) to the JSON dependency file that was created by `python_package_analyze`. The `environment-name` argument specifies the name for the environment that is created. 

## OPTIONS

-h        Show this help message

## EXIT STATUS

On success, returns zero. On failure, returns non-zero.
- 1 - Invalid command format
- 2 - Invalid path to the JSON dependency file

## EXAMPLE

An dependencies file `dependencies.json` contains the following:

`{"python": "3.7.3", "modules": ["antigravity", "matplotlib"]}`

To generate a Conda environment with the Python 3.7.3 interpreter and the `antigravity` and `matplotlib` modules preinstalled and with name `example_venv`, run the following command:

`$ python_package_create dependencies.json example_venv`

Note that this will also create an `example_venv.tar.gz` environment tarball within the current working directory, which can then be exported to different machines for execution.

## POSSIBLE IMPROVEMENTS
1. Figure out alternative to using `subprocess.call()` to create the Conda environment (perhaps make a Bash script altogether)
- Most of the execution occurs within the subprocess call, so basically a Bash script, but easier to use Python to parse the JSON file and write to the requirement file
- Perhaps use a JSON parsing command line utility within Bash script instead, such as `jq`
- If a Conda environment API for Python is ever created, it would be very useful here, as we could remove the subprocess call completely
2. Remove redirection all output to `/dev/null`
- All output from the subprocess call is removed for organization purposes, but some commands like `pip install` might be useful for the user to see
- Removing redirection also makes it much easier to debug



# python_package_run(1)

## NAME

`python_package_run` - wrapper script that executes Python script within an isolated Conda environment

## SYNOPSIS

`python_package_run [options] <environment-name> <python-command-string>`

## DESCRIPTION

The `python_package_run` tool acts as a wrapper script for a Python task, running the task within the specified Conda environment. `python_package_run` can be utilized on different machines within the Work Queue system to unpack and activate a Conda environment, and run a task within the isolated environment.

The `environment-name` argument is the name of the Conda environment in which to run the Python task. The `python-command-string` is full shell command of the Python task that will be run within the Conda environment.

## OPTIONS 

-h        Show this help message

## EXIT STATUS

On success, returns 0. On failure, returns non-zero.
1 - Invalid command format
2 - Environment tarball does not exist
3 - Failed trying to extract environment tarball
4 - Failed trying to activate Conda environment
(Note: The wrapper script captures the exit status of the Python command string. It is possible that the exit code of the Python task overlaps with the exit code of the wrapper script)

## EXAMPLE

A Python script `example.py` has been analyzed using `python_package_analyze` and a corresponding Conda environment named `example_venv` has been created, with all the necessary dependencies preinstalled. To execute the script within the environment, run the following command:

`python_package_run example.py "python3 example.py"`

This will run the `python3 example.py` task string within the activated `example_venv` Conda environment. Note that this command can be performed either locally, on the same machine that analyzed the script and created the environment, or remotely, on a different machine that contains the Conda environment tarball and the `example.py` script.

## POSSIBLE IMPROVEMENTS
1. Do protection checking against dangerous shell commands, as the script runs the command line argument directly
- The program directly runs the task string that is passed in, which means the user could send a task that is harmful to the worker machine
- Perhaps WorkQueue already uses protection checking for the task strings, in which case it is not necessary




# HOW TO TEST OVERALL FUNCTIONALITY

Desired Python script to run: `hi.py`

1. `./python_package_analyze hi.py output.json`
- Generates the appropriate JSON file in the current working directory
2. `./python_package_create output.json venv`
- Will create a Conda environment in the Conda `envs` folder, and will create a packed tarball of the environment named `venv.tar.gz` in the current working directory
- To more easily debug, remove the redirected output to `/dev/null` in the subprocess call to see all output of the environment creation and module installation
3. `./python_package_run venv "python3 hi.py"`
- Runs the `python3 hi.py` task command within the activated `venv` Conda environment