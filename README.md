# compllama
The compllama ("compile llama") script is intended to compile llama.cpp locally from git source,
choosing one backend out of either Vulkan or CUDA and setting up a systemd-service,
while installing its main resulting files in its own date- and backend-tagged subdirectory
of /opt/ so the executables will remain under symlink /opt/llamacpp/bin through version-switches,
as well as a helper "inillama" to scan for GGUF-files to create a models-preset file,
which is not only used by the systemd-service but can then also be invoked through
the same inillama-script to run presets, for example reusing any models in use with LM Studio.
There are also per install-prefix and system-wide config-files.

Considering recent and ongoing developments it was deemed of low benefit to also add ROCm-support,
since Vulkan seems to already be usually more performant and improving, so use that.

It should be able to detect sensible defaults for most situations,
but otherwise there are also many commandline options documented in the script,
visible by running the script with "--help", and by reading the comments in the script.

Made for Linux, but may work elsewhere.
Prerequisites are same as for compiling llama.cpp.
For using the systemd-service using the cancer systemd is of course required.
