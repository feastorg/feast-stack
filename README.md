# feast-stack

Deployment and orchestration for the FEAST stack.

## Dev Setup

For getting started with development on a new Linux machine:

```sh
sudo apt update

sudo apt install -y \
build-essential \
cmake \
ninja-build \
pkg-config \
git \
curl \
zip \
unzip \
tar

cd ~
git clone https://github.com/microsoft/vcpkg ~/vcpkg
~/vcpkg/bootstrap-vcpkg.sh
```
