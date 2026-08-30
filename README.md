# ORB-SLAM3 for Ubuntu 24.04 LTS

This repository contains ORB-SLAM3 with fixes and modifications for compatibility with Ubuntu 24.04 LTS (Noble Numbat).

## System Requirements

- **OS**: Ubuntu 24.04 LTS
- **Compiler**: GCC 13.3.0, G++ 13.3.0
- **CMake**: 3.28.3 or higher
- **Memory**: At least 4GB RAM recommended
- **Storage**: ~2GB for dependencies and build files

## Prerequisites

### 1. Install Build Tools

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential cmake git pkg-config
```

### 2. Install Core Dependencies

```bash
# Eigen3 (Linear algebra library)
sudo apt install -y libeigen3-dev

# OpenCV (Computer vision library)
sudo apt install -y libopencv-dev

# Pangolin dependencies
sudo apt install -y libglfw3-dev libgl1-mesa-dev libglu1-mesa-dev libepoxy-dev

# Boost serialization (required by DBoW2 / the main library)
sudo apt install -y libboost-serialization-dev
```

> **Check your OpenCV version before building** — this matters more than it sounds:
>
> ```bash
> pkg-config --modversion opencv4
> ```
>
> The version-pinned `find_package(OpenCV X.Y ...)` calls in `CMakeLists.txt` and `Thirdparty/DBoW2/CMakeLists.txt` must match (or be compatible with) whatever `libopencv-dev` actually installed on your system — Ubuntu 24.04 has shipped different OpenCV point releases (e.g. 4.5.4, 4.6.0) depending on when packages were pulled. If your version doesn't match the pin in those two files, `find_package` silently fails to find OpenCV and you'll see `OpenCV > 3.0 not found` (from DBoW2) or a similar fatal error from the top-level CMakeLists.txt. See [Troubleshooting](#troubleshooting) below for the fix.

### 3. Install Optional Dependencies

#### For RealSense Camera Support

If you plan to use Intel RealSense cameras, build librealsense from source:

```bash
# Remove apt version if installed
sudo apt remove librealsense2-dev librealsense2-utils

# Install build dependencies
sudo apt install -y libusb-1.0-0-dev libgtk-3-dev

# Clone and build librealsense
cd ~
git clone https://github.com/IntelRealSense/librealsense.git
cd librealsense
git checkout v2.55.1

# NOTE: if you're re-running this and a build/ directory already exists
# from a previous attempt, `mkdir build` will fail and `cd build` will
# silently never run (because of `&&`), which then makes every command
# after this operate on the wrong directory. Use one of:
rm -rf build && mkdir build && cd build
# or, if the directory is already there and safe to reuse:
# mkdir -p build && cd build

cmake .. \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_WITH_DDS=false \
    -DBUILD_EXAMPLES=true \
    -DBUILD_GRAPHICAL_EXAMPLES=true

make -j$(nproc)
sudo make install
sudo ldconfig

# Setup udev rules (host only — see note below if building in a container)
sudo cp ../config/99-realsense-libusb.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules && sudo udevadm trigger
```

> **Building inside a container?** The udev rules step above only makes sense on a real Linux host with udev/systemd running. Most containers don't have `/etc/udev` at all, and `udevadm` won't be installed — that's expected, not a bug. Handle device permissions on the **host**, and pass the RealSense device through to the container at startup (e.g. `-v /dev/bus/usb:/dev/bus/usb` or `--privileged` for testing), then skip the udev step inside the container entirely.

## Installation

### 1. Setup Directory Structure

```bash
# Create workspace
cd ~
mkdir orb-slam3-root && cd orb-slam3-root

# Set environment variable (add to ~/.bashrc for persistence)
export ORB_SLAM3_ROOT_PATH=~/orb-slam3-root
echo "export ORB_SLAM3_ROOT_PATH=~/orb-slam3-root" >> ~/.bashrc
```

### 2. Build Pangolin

```bash
cd $ORB_SLAM3_ROOT_PATH

# Clone Pangolin
git clone https://github.com/stevenlovegrove/Pangolin.git
cd Pangolin
git checkout v0.9.2

# Build
mkdir build && cd build
cmake ..
make -j$(nproc)
```

**Test Pangolin** (optional):

```bash
$ORB_SLAM3_ROOT_PATH/Pangolin/build/examples/HelloPangolin/HelloPangolin
```

### 3. Clone This Repository

```bash
cd $ORB_SLAM3_ROOT_PATH
git clone <YOUR_FORKED_REPO_URL> ORB-SLAM3
cd ORB-SLAM3
```

### 4. Build ORB-SLAM3

```bash
chmod +x build.sh
./build.sh
```

The build process will:

- Compile DBoW2 (vocabulary management)
- Compile g2o (graph optimization)
- Compile Sophus (Lie algebra)
- Uncompress vocabulary file
- Build ORB-SLAM3 library and examples

### 5. Install Sophus Globally

This step is required if you plan to use ORB-SLAM3 with ROS2:

```bash
cd $ORB_SLAM3_ROOT_PATH/ORB-SLAM3/Thirdparty/Sophus/build
sudo make install
```

To uninstall later:

```bash
sudo rm -rf /usr/local/include/sophus
sudo rm -rf /usr/local/share/sophus
```

## Testing

### Option 1: Test with Webcam

```bash
cd $ORB_SLAM3_ROOT_PATH/ORB-SLAM3
./Examples/Monocular/mono_tum Vocabulary/ORBvoc.txt Examples/Monocular/TUM1.yaml
```

Move your camera around. You should see feature tracking and map building in real-time.

### Option 2: Test with EuRoC Dataset

Download the MH_01_easy sequence:

```bash
cd $ORB_SLAM3_ROOT_PATH/ORB-SLAM3
mkdir datasets && cd datasets

# Download from ETH Research Collection
wget https://www.research-collection.ethz.ch/bitstream/handle/20.500.11850/263889/1/MH_01_easy.zip

# Extract
unzip MH_01_easy.zip
mkdir MH01
mv mav0 MH01/mav0
cd ..

# Run test
./Examples/Monocular/mono_euroc \
    Vocabulary/ORBvoc.txt \
    Examples/Monocular/EuRoC.yaml \
    datasets/MH01 \
    Examples/Monocular/EuRoC_TimeStamps/MH01.txt \
    dataset-MH01_mono
```

### Option 3: Test with TUM RGB-D Dataset

```bash
cd $ORB_SLAM3_ROOT_PATH/ORB-SLAM3/datasets
mkdir TUM && cd TUM

# Download a sequence
wget https://cvg.cit.tum.de/rgbd/dataset/freiburg1/rgbd_dataset_freiburg1_xyz.tgz
tar -xvf rgbd_dataset_freiburg1_xyz.tgz

cd $ORB_SLAM3_ROOT_PATH/ORB-SLAM3

# Run RGB-D example
./Examples/RGB-D/rgbd_tum \
    Vocabulary/ORBvoc.txt \
    Examples/RGB-D/TUM1.yaml \
    datasets/TUM/rgbd_dataset_freiburg1_xyz \
    datasets/TUM/associations/fr1_xyz.txt
```

## Available Examples

After successful build, you'll find these executables in `Examples/`:

### Monocular

- `mono_euroc` - EuRoC MAV dataset
- `mono_kitti` - KITTI dataset
- `mono_tum` - TUM monocular
- `mono_tum_vi` - TUM VI dataset

### Monocular-Inertial

- `mono_inertial_euroc` - EuRoC with IMU
- `mono_inertial_tum_vi` - TUM VI with IMU

### Stereo

- `stereo_euroc` - EuRoC stereo
- `stereo_kitti` - KITTI stereo
- `stereo_tum_vi` - TUM VI stereo

### Stereo-Inertial

- `stereo_inertial_euroc` - EuRoC stereo with IMU
- `stereo_inertial_tum_vi` - TUM VI stereo with IMU

### RGB-D

- `rgbd_tum` - TUM RGB-D dataset

## Key Modifications for Ubuntu 24.04

This repository includes the following fixes for Ubuntu 24.04 compatibility:

1. **CMakeLists.txt** (line 33):

   ```cmake
   find_package(OpenCV 4.6)  # Match this to your installed OpenCV version — see Troubleshooting
   ```

2. **CMakeLists.txt** (line 41):

   ```cmake
   find_package(Eigen3 REQUIRED)  # Version number removed
   ```

3. **ThirdParty/DBoW2/CMakeLists.txt** (line 32):

   ```cmake
   find_package(OpenCV 4.6 QUIET)  # Match this to your installed OpenCV version — see Troubleshooting
   ```

4. **ThirdParty/Sophus/CMakeLists.txt** (line 23):

   ```cmake
   SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Werror -Wextra -std=c++11 -Wno-error=array-bounds -Wno-deprecated-declarations -ftemplate-backtrace-limit=0")
   ```

5. **ThirdParty/Sophus/CMakeLists.txt** (line 35):

   ```cmake
   find_package(Eigen3 REQUIRED)  # Version number removed
   ```

6. **Examples/Monocular/mono_euroc.cc** (line 83):

   ```cpp
   ORB_SLAM3::System SLAM(argv[1],argv[2],ORB_SLAM3::System::MONOCULAR, true);
   // Viewer enabled by default
   ```

## Environment Variables

Add these to your `~/.bashrc` for convenience:

```bash
# ORB-SLAM3 root path
export ORB_SLAM3_ROOT_PATH=~/orb-slam3-root

# Add library paths (if needed)
export LD_LIBRARY_PATH="$ORB_SLAM3_ROOT_PATH/ORB-SLAM3/lib:$LD_LIBRARY_PATH"
export LD_LIBRARY_PATH="$ORB_SLAM3_ROOT_PATH/Pangolin/build:$LD_LIBRARY_PATH"
```

## Troubleshooting

### Build Errors

**Problem**: `error: 'array' was not declared in this scope`
**Solution**: Already fixed in this repository. Ensure you're using the modified Sophus CMakeLists.txt.

**Problem**: `CMake Error: OpenCV > 3.0 not found.` (from `Thirdparty/DBoW2/CMakeLists.txt`) or a similar OpenCV-not-found error from the top-level `CMakeLists.txt`
**Solution**: This almost always means the OpenCV version pin in `find_package(OpenCV X.Y ...)` doesn't match what's actually installed. Check your installed version:

```bash
pkg-config --modversion opencv4
```

Then edit the version number in **both**:
- `CMakeLists.txt` (`find_package(OpenCV 4.6)`)
- `Thirdparty/DBoW2/CMakeLists.txt` (`find_package(OpenCV 4.6 QUIET)`)

to match, e.g. `find_package(OpenCV 4.5 QUIET)` for OpenCV 4.5.4 — or drop the version number entirely (`find_package(OpenCV QUIET)`) to accept any OpenCV 3+/4.x.

**Problem**: `fatal error: boost/serialization/serialization.hpp: No such file or directory`
**Solution**: Boost's serialization headers aren't installed. Run:

```bash
sudo apt install -y libboost-serialization-dev
```

then re-run `./build.sh`.

**Problem**: RealSense warnings about fastcdr/fastrtps
**Solution**: This is harmless if you don't use RealSense. To fix completely, rebuild librealsense with `-DBUILD_WITH_DDS=false` (see Prerequisites section).

**Problem**: `mkdir: cannot create directory 'build': File exists` followed by CMake, make, and `cp` errors that all look like they're operating on the wrong directory
**Solution**: This happens when re-running a `mkdir build && cd build` command after `build/` already exists from a previous attempt — `mkdir` fails, so (because of `&&`) `cd build` never runs, and every following command executes one directory up. Use `rm -rf build && mkdir build && cd build` to reset, or `mkdir -p build && cd build` if you just want to reuse the existing directory.

### Runtime Errors

**Problem**: Segmentation fault when loading vocabulary
**Solution**: Ensure you've extracted the vocabulary:

```bash
cd $ORB_SLAM3_ROOT_PATH/ORB-SLAM3/Vocabulary
tar -xf ORBvoc.txt.tar.gz
```

**Problem**: Camera not detected
**Solution**:

```bash
# Check camera device
ls -l /dev/video*

# Test with v4l2
v4l2-ctl --list-devices
```

If you're running inside a container, make sure the camera/USB device was actually passed through at container start (see the RealSense note in Prerequisites) — a missing device mapping looks identical to a driver problem from inside the container.

**Problem**: Black screen/no tracking
**Solution**:

- Ensure adequate lighting
- Move camera slowly at first
- Point at textured surfaces (not blank walls)
- Check that images are arriving (press 'v' in viewer to toggle views)

## Performance Tips

1. **CPU Usage**: ORB-SLAM3 is CPU-intensive. For best performance:
   - Close unnecessary applications
   - Use `make -j$(nproc)` to utilize all cores during build

2. **Memory**: The vocabulary file loads into RAM (~80MB). Ensure sufficient memory.

3. **Real-time Performance**:
   - Monocular: 30 FPS on modern CPUs
   - Stereo: 20-30 FPS
   - RGB-D: 30 FPS

## Citation

If you use ORB-SLAM3 in your research, please cite:

```bibtex
@article{ORBSLAM3_TRO,
  title={{ORB-SLAM3}: An Accurate Open-Source Library for Visual, Visual-Inertial
         and Multi-Map {SLAM}},
  author={Campos, Carlos AND Elvira, Richard AND Rodriguez, Juan J. G\'omez AND
          Montiel, Jos\'e M. M. AND Tard\'os, Juan D.},
  journal={IEEE Transactions on Robotics},
  volume={37},
  number={6},
  pages={1874-1890},
  year={2021}
}
```

## License

ORB-SLAM3 is released under a [GPLv3 license](LICENSE).

## Acknowledgments

- Original ORB-SLAM3 by Carlos Campos et al.
- Ubuntu 24.04 compatibility fixes
- Community contributions for stereo fixes

## Support

For issues specific to Ubuntu 24.04 compatibility, please open an issue in this repository.

For general ORB-SLAM3 questions, refer to the [original repository](https://github.com/UZ-SLAMLab/ORB_SLAM3).

## Related Resources

- [ORB-SLAM3 Paper](https://arxiv.org/abs/2007.11898)
- [Pangolin Documentation](https://github.com/stevenlovegrove/Pangolin)
- [EuRoC Dataset](https://projects.asl.ethz.ch/datasets/euroc-mav/)
- [TUM RGB-D Dataset](https://cvg.cit.tum.de/data/datasets/rgbd-dataset)
