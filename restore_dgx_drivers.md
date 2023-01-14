# How to fix the DGX 1 in case there are driver problems

Over the last year, I have had to restore the DGX1 multiple times due to driver issues after updating some packages, trying to install correct CUDA versions for a specific version of tensorflow, or messing up stuff after fiddling with kernel internals, etc.

It's always pretty stressful and frustrating, and I always forget what worked the last time, so I've gathered the process that fixed it most recently for future reference.

## nvidia-smi has disappeared 

This latest accident has no precisely known cause, but after trying to recompile python3 with dtrace support to trace the inner workings of the BERT workload (following this guide https://ish-ar.io/python-ebpf-tracing/), suddenly, `nvidia-smi` was not on the system anymore!

When trying to run it, we get a message suggesting us to install a nvidia driver suite, and multiple options. 

What followed was installing the wrong drivers apparently, leading the DGX to request a version of cuda, then trying to install that with no success. Anyway, after rebooting a few times (which sometimes solves the issue) I had to manually remove all previously installed drivers, etc. and re-install correct versions.

The instructions found [here](https://forums.developer.nvidia.com/t/nvidia-smi-has-failed-because-it-couldnt-communicate-with-the-nvidia-driver-make-sure-that-the-latest-nvidia-driver-is-installed-and-running/197141) put me on the right track:

Remove everything related to NVIDIA 
```
sudo apt-get remove --purge '^nvidia-.*'
sudo apt-get remove --purge '^libnvidia-.*'
sudo apt-get remove --purge '^cuda-.*'
```
Then reinstall the kernel headers (I had played with that which may have been the source of the problem)
```
sudo apt-get install linux-headers-$(uname -r)
```

Then, I followed the official instructions found [here](https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=20.04&target_type=deb_local) after choosing the correct system options (x86_64, ubuntu 20.04).

```
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/cuda-ubuntu2004.pin
sudo mv cuda-ubuntu2004.pin /etc/apt/preferences.d/cuda-repository-pin-600
wget https://developer.download.nvidia.com/compute/cuda/12.0.0/local_installers/cuda-repo-ubuntu2004-12-0-local_12.0.0-525.60.13-1_amd64.deb
sudo dpkg -i cuda-repo-ubuntu2004-12-0-local_12.0.0-525.60.13-1_amd64.deb
sudo cp /var/cuda-repo-ubuntu2004-12-0-local/cuda-*-keyring.gpg /usr/share/keyrings/
sudo apt-get update
sudo apt-get -y install cuda
```
There were some errors related to public keys, but it worked out in the end!

Finally, the purging had removed nvidia-docker, which is necessary to run the workloads in docker containers. I had to reinstall it using:

```
sudo apt install -y nvidia-docker2
sudo systemctl daemon-reload
sudo systemctl restart docker
```

So don't be scared of purging all the drivers. 