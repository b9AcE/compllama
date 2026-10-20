# compllama
The compllama ("compile llama") script is intended to compile llama.cpp locally from git source,<br />
choosing one backend out of either Vulkan or CUDA and setting up a systemd-service,<br />
while installing its main resulting files in its own date- and backend-tagged subdirectory<br />
of /opt/ so the executables will remain under symlink /opt/llamacpp/bin through version-switches,<br />
as well as a helper "inillama" to scan for GGUF-files to create a models-preset file,<br />
which is not only used by the systemd-service but can then also be invoked through<br />
the same inillama-script to run presets, for example reusing any models in use with LM Studio.<br />
There are also per install-prefix and system-wide config-files.<br />
Everything is of course written so that it can be run repeatedly on the same computer,<br />
e.g. only touching parts of potentially shared config-files that are marked as written by compllama,<br />
and the uninstall-option checks for the same markers,<br />
and inillama only appends to the presets-file unless explicitly user-requested.<br />
<br />
Considering recent and ongoing developments it was deemed of low benefit to also add ROCm-support,<br />
since Vulkan seems to already be usually more performant and improving, so use that.<br />
<br />
It should be able to detect sensible defaults for most situations,<br />
but otherwise there are also many commandline options documented in the script,<br />
visible by running the script with "--help", and by reading the comments in the script.<br />
<br />
Made for Linux, but may work elsewhere.<br />
Prerequisites are same as for compiling llama.cpp.<br />
For using the systemd-service using the cancer systemd is of course required.<br />
