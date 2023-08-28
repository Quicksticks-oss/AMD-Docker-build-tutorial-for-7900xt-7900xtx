# AMD-Docker-build-tutorial-for-7900xt-7900xtx

[Link](https://github.com/AUTOMATIC1111/stable-diffusion-webui/discussions/8172#discussion-4904543)

## Hello Team,

After purchasing my first AMD GPU (7900xt) I hit a big stop when I tried to run SD on my new config. After some troubleshooting/reading lot of github issues, I was able to build an unstable (really ! very unstable) docker image with 7900xt (gfx1100 GPU) support for ROCM and Pytorch.

If you want to give a try, here is the actual workflow to build this docker image.

Note: you'll need to have at least up to 40GB of freespace.
Note 2: I am no docker specialist, nor pytorch. This workflow can be simplified, however as it's a community discussion, I hope someone more knowledgeable than me could simplified it

I prefer to re-write: at the moment it's very buggy and unstable so be prepare to have issues.

If you want to try by yourself, here is how I build my image:

Git clone pytorch repository:
``git clone --recursive https://github.com/pytorch/pytorch``

In the ``.ci/docker/`` there is a file named ``build.sh``, at line 203, you'll have the following esac statement:

Just after the ;; Add the following block:
```
  pytorch-linux-focal-rocm543-n-py3)
    ANACONDA_PYTHON_VERSION=3.10
    GCC_VERSION=9
    PROTOBUF=yes
    DB=yes
    VISION=yes
    ROCM_VERSION=5.4.3
    NINJA_VERSION=1.9.0
    CONDA_CMAKE=yes
    PYTORCH_ROCM_ARCH=gfx1100
    IMAGE_NAME=pytorch-linux-focal-rocm543-py3.8
```
So the complete output starting at line 194 is:
```
  pytorch-linux-focal-rocm-n-py3)
    ANACONDA_PYTHON_VERSION=3.8
    GCC_VERSION=9
    PROTOBUF=yes
    DB=yes
    VISION=yes
    ROCM_VERSION=5.4.3
    NINJA_VERSION=1.9.0
    CONDA_CMAKE=yes
    ;;
  pytorch-linux-focal-rocm543-n-py3)
    ANACONDA_PYTHON_VERSION=3.10
    GCC_VERSION=9
    PROTOBUF=yes
    DB=yes
    VISION=yes
    ROCM_VERSION=5.4.3
    NINJA_VERSION=1.9.0
    CONDA_CMAKE=yes
    PYTORCH_ROCM_ARCH=gfx1100
    IMAGE_NAME=pytorch-linux-focal-rocm543-py3.8
    ;;
```
Save your file.
3. Now in the uubntu-rocm subfolder of .ci/docker/ you'll have the Dockerfile that will be used to build your image. You'll have to add three lines at the bottom of this file:
```
WORKDIR /tmp
RUN git clone --recursive https://github.com/pytorch/pytorch.git && cd pytorch && .ci/pytorch/build.sh
RUN git clone https://github.com/pytorch/vision.git && cd vision && FORCE_CUDA=1 python setup.py install
```
Save your file.

I think that part can be optimized to reduce docker image size
4. Now go back to your previous folder and launch the following command (I assume your user is in the docker group):
```
./build.sh pytorch-linux-focal-rocm543-n-py3
```
The build process will be launched (it takes up to 40 minutes on my computer).
5. If the build process finished correctly, you'll have a huge image:
```
$ docker images
REPOSITORY             TAG       IMAGE ID       CREATED              SIZE
tmp.7iqw2ylj9q         latest    005ef5546f67   About a minute ago   35.9GB
rocm/pytorch-nightly   latest    2cfd54c9e428   2 days ago           38.2GB
Time to tag the image named tmp.7iqw2ylj9q with a more human readable name:
```
```
docker image tag 005ef5546f67 localbuild-rocm:latest
```
Once done, you can run your container with a mounted folder that target an empty folder (we will recreate from scratch the python venv) on your host (replace /home/YOURUSER/stable-diffusion-webui/ by your username):
```
docker run -it --device=/dev/kfd --device=/dev/dri --group-add=video --ipc=host --cap-add=SYS_PTRACE --security-opt seccomp=unconfined -p 127.0.0.1:7860:7860 -v /home/YOURUSER/stable-diffusion-webui/:/var/lib/jenkins/stable-diffusion-webui/ localbuild-rocm:latest
```
Once connected in your container, let's reassign group ownership on some important devices:
```
sudo chgrp video /dev/kfd
```
Now, there is an additional device that could differs from my environment, so you'll have to replace the /dev/dri/renderXXX by the one mounted in your docker container.
```
sudo chgrp video /dev/dri/renderXXX
```
Now, if you run rocminfo you must see your device detected, if not, no need to continue, you won't be able to run SD.
Example:
```
jenkins@f56a43abba09:~/stable-diffusion-webui$ rocminfo
ROCk module is loaded
=====================    
HSA System Attributes    
=====================    
Runtime Version:         1.1
System Timestamp Freq.:  1000.000000MHz
Sig. Max Wait Duration:  18446744073709551615 (0xFFFFFFFFFFFFFFFF) (timestamp count)
Machine Model:           LARGE                              
System Endianness:       LITTLE                             
                      

[...]

*******                  
Agent 2                  
*******                  
  Name:                    gfx1100                            
  Uuid:                    GPU-XX                             
  Marketing Name:                                             
  Vendor Name:             AMD
```
If step 9 is OK, let's test with pytorch to confirm that it's also working, run the python3 interpreter and type the following code:
```
import torch
```
It will load torch python module, then to check if torch is able to work your GPU, you can use the following command (note: the >>> should not be typed):
```
>>> torch.cuda.is_available()
True
>>> torch.cuda.current_device()
0
>>> torch.cuda.get_device_name(0)
'Radeon RX 7900 XT'
```
If your GPU appears, you're good to go to the next step.
11. Let's git clone stable diffusion repo in your $HOMEDIR (in my case /var/lib/jenkins) :
```
git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui.git stable-diffusion-webui/
```
Once clone, we will build SD venv based on the compiled version from this docker image:
```
python3 -m venv --copies --system-site-packages venv
```
To check if everything is correct, if you try to launch:
```
pip list
```
You'll have a very long list of python packages, that list, should be identical in SD's venv, to confirm that, source SD's venv
```
source venv/bin/activate
```
Then, if you use the same command as above pip list you must have the same content, if it's identical, you can go to next and final step : updating sd to listen on 0.0.0.0
14. Edit webui-user.sh and replace the following line (line 13):
```
#export COMMANDLINE_ARGS=""
```
by
```
export COMMANDLINE_ARGS="--listen"
```
(note: if you want to install extension, I found that option  --enable-insecure-extension-access should be added to this command line too).
15. All modifications are done, now it's time to start SD with ./webui.sh

Regards,

Nikos

Posted here by: Miles Ryan
Created by: Nikos
