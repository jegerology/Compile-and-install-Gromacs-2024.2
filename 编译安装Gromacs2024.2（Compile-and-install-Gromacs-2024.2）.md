*视频链接：【Gromacs2024.2版安装全程及代码分享-纯干货】 https://www.bilibili.com/video/BV1GM4m1m7uq/?share_source=copy_web&vd_source=e986a18a26b36d1763a9c84f951a2ff0，by 大导心腹*
# 准备工作-_Preparatory work_
- **声明：编译安装过程使用系统为 centos7，服务器编译环境及硬件配置：cmake 3.29.6/gcc 9.3.1/cuda 12.4/openmpi 5.0.1/fftw为 gromacs 自行安装（CPU使用了fftw-3.3.8版本，GPU使用了cuFFT）/两颗Intel(R) Xeon(R) CPU E5-2699 v3 @ 2.30GHz/NVIDIA GeForce RTX 4060 Ti/64GB 内存。值得一提的是，root用户最好预留足够的根分区储存空间，以免模拟过程中产生的文件积攒导致空间不足而任务失败。**
- Declaration: This compilation process uses the centos7 system, so check whether the compilation environment used is upgraded, and the preparation part needs to operate according to their actual system environment and its own gcc, cmake and other versions.
## 检查系统编译环境-_Check the system compilation environment_

- **提前检查编译环境及系统信息，如 cuda 版本、gpu 驱动版本 、SIMD 等，**
- Check system information in advance, such as the cuda version, gpu driver version, and SIMD
```
mpirun --version
cmake --version
gcc --version
nvcc --version
fftw-wisdom --version #It can be installed by Gromacs itself
lscpu  # check CPU information, such as SIMD
nvidia-smi # check nvidia driver and cuda
```
## 升级 GCC-_Upgrade GCC_
```
sudo yum -y install centos-release-scl
sudo yum install devtoolset-9-gcc devtoolset-9-gcc-c++ devtoolset-9-binutils
scl enable devtoolset-9 bash
echo "source /opt/rh/devtoolset-9/enable" >> ~/.bashrc
gcc --version
```
## 升级 CMake-_Upgrade CMake_
```
sudo yum install openssl-devel
sudo yum remove cmake3  # Uninstall the current cmake3
wget https://cmake.org/files/v3.29/cmake-3.29.6.tar.gz
tar -zxvf cmake-3.29.6.tar.gz
cd cmake-3.29.6
./bootstrap
make 
sudo make install
```
## 安装 OpenMPI-_Install the OpenMPI_
```
wget https://download.open-mpi.org/release/open-mpi/v5.0/openmpi-5.0.1.tar.bz2
tar xf openmpi-5.0.1.tar.bz2
cd openmpi-5.0.1
./configure --prefix=/usr/local/openmpi 
make -j $(nproc) all
sudo make install 
echo 'export PATH=/usr/local/openmpi/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
mpirun --version
ompi_info #详细信息
```
## （安装**快速傅立叶变换库**FFTW-_Install FFTW）_
```
sudo yum remove fftw 
wget http://www.fftw.org/fftw-3.3.10.tar.gz 
tar -xvf fftw-3.3.10.tar.gz 
cd fftw-3.3.10 
./configure --enable-float --enable-sse2 --enable-avx2 --prefix=/jer/GMX2024.2/fftw3310/fftw-3.3.10  #改为实际路径
make 
sudo make install
echo "export LD_LIBRARY_PATH=/jer/GMX2024.2/fftw3310/fftw-3.3.10/lib:\$LD_LIBRARY_PATH" >> ~/.bashrc  # 如果安装到了非标准路径，更新环境变量，将 FFTW 的库路径添加到 LD_LIBRARY_PATH
source ~/.bashrc
```
## 重装 cuda toolkit-_Reinstall the cuda toolkit_
```
# 经nvidia-smi与nvcc --version检查发现两者版本不同，在CMake编译时会报错，笔者重新安装适配的cuda toolkit 12.4
# 卸载旧版cuda toolkit
sudo yum remove cuda
sudo yum autoremove
# 清理旧版本的cuda文件
sudo yum remove "*cublas*" "cuda*" 
sudo yum remove "*nvidia*"  
sudo yum autoremove 
# 安装
sudo yum-config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel7/x86_64/cuda-rhel7.repo  # 添加 CUDA 12.4 的 YUM 源
sudo yum clean all  # 清除 YUM 缓存
sudo yum -y install cuda-toolkit-12-4  # 安装 CUDA Toolkit 12.4，笔者在此耗时半小时左右
echo -e "export PATH=/usr/local/cuda-12.4/bin:\$PATH\nexport LD_LIBRARY_PATH=/usr/local/cuda-12.4/lib64:\$LD_LIBRARY_PATH" >> ~/.bashrc && source ~/.bashrc && nvcc --version
```
# 下载 Gromacs 源代码-_Download the Gromacs source code_
```
#最好提前 mkdir 特定目录，比如: 用户名/GMX/GMX2024.2
wget https://ftp.gromacs.org/gromacs/gromacs-2024.2.tar.gz
wget https://ftp.gromacs.org/regressiontests/regressiontests-2024.2.tar.gz #测试包
tar -xvzf gromacs-2024.2.tar.gz
tar -xvzf regressiontests-2024.2.tar.gz
```
# Gromacs 的编译与安装-_Compile and install Gromacs_

- **若有多个版本的 Gromacs 则需要分开安装，**
- **笔者所用 NVIDIA GeForce RTX 4060 Ti GPU 支持的 CUDA 架构是 compute_86，在配置 GROMACS 时，需要确保 CMake 配置中指定的 GPU 架构与 GPU 兼容。由于笔者本次遇到了关于 ‘compute_35’ 的错误，需要更新 CMake 配置以匹配 GPU：**
   1. **编辑 CMake 配置：打开 GROMACS CMake 配置文件，包括源代码目录中的 CMakeLists.txt 和构建目录中的 CMakeCache.txt。**
   2. **修改 GPU 架构设置：找到所有提及 ‘compute_35’ 的地方，并将其改为 ‘compute_86’（ 笔者操作时 CMakeCache.txt 中有三处， CMakeLists.txt 没有）。**
   3. **清理构建目录：在您的构建目录中运行以下命令来清理之前的构建文件：`make clean`，或者直接删除 build 目录，重新 cmake 编译。**
- **另外，centos7 默认 python 3.6，但 Gromacs 需要 python >=3.7，而且需要基于 3.7 安装Sphinx：首先安装 python3.7，然后设为 python3 默认版本，利用 3.7 的 pip 进行Sphinx安装。**
- If there are multiple versions of Gromacs, they need to be installed separately
- The CUDA architecture supported by NVIDIA GeForce RTX 4060 Ti Gpus is compute_86. When configuring GROMACS, ensure that the GPU architecture specified in the CMake configuration is compatible with the GPU. Since I encountered an error with 'compute_35' this time, I need to update the CMake configuration to match the GPU:
   1. undefined edit the CMake configuration: open the GROMACS CMake configuration file, including CMakeLists.txt in the source directory and CMakeCache.txt in the build directory.
   2. undefined Modify GPU architecture Settings: find all references to 'compute_35' and change them to 'compute_86' (CMakeCache.txt has three, CMakeLists.txt has no error ).
   3. undefined clean build directory: Run the following command in your build directory to clean up the previous build file: make clean, or simply delete the build directory and re-cmake compilation
- In addition, centos7 defaults to python 3.6, but Gromacs requires python >=3.7, and needs to install Sphinx based on3.7, first install python3.7, and then set to python3 default. Use 3.7 pip for Sphinx installation
```
sudo update-alternatives --install /usr/bin/python3 python3 /usr/local/bin/python3.7 1 #check your path
sudo update-alternatives --config python3
python3 --version
echo "alias python3='/usr/local/bin/python3.7'" >> ~/.bashrc
source ~/.bashrc
# 由于yum依赖python3.6
sudo ln -sf /usr/bin/python3.6 /usr/bin/python #保证yum正常
/usr/local/bin/python3.7 -m ensurepip --upgrade 
/usr/local/bin/python3.7 -m pip install -U sphinx
```
```
#rm -rf build  #删除错误编译的build，重新cmake编译
mkdir build #创建一个构建目录 build 并运行 CMake 来配置编译选项。
cd build
cmake ../gromacs-2024.2 \
  -DCMAKE_C_COMPILER=gcc \
  -DPYTHON_EXECUTABLE=/usr/local/bin/python3.7 \
  -DPython3_ROOT_DIR='/usr/local/bin' \
  -DGMX_SIMD=AVX2_256 \
  -DGMX_MPI=OFF \ # 编译时报错未解决索性关闭
  -DGMX_GPU=CUDA \
  -DCMAKE_INSTALL_PREFIX=~/Software/GMX/2024.2-cuda \
  -DGMX_BUILD_OWN_FFTW=ON \
  -DREGRESSIONTEST_DOWNLOAD=OFF \
  -DREGRESSIONTEST_PATH=../regressiontests-2024.2 \
  -DGMX_CUDA_TARGET_SM="86" \
  -DGMX_CUDA_TARGET_COMPUTE="86"
```
```
make -j $(nproc) # 约用时4min
make check  # 约用时20min
make install
```
# 配置环境变量-_Configuring environment variables_

- **若有多个版本的 gmx 还需要改名以便调用特定版本的 gmx**
- If you have more than one version of gmx, you also need to change the name in order to call a specific version of gmx
```
echo -e 'export PATH=/jer/GMX2024.2/build/bin:$PATH\nalias gmx2024="gmx"' >> ~/.bashrc && source ~/.bashrc
# 其中/jer/GMX2024.2/build/bin应改为自己实际路径
gmx2024 --version 
```
# 性能测试-_Performance test_

- **运行测试案例，确保 GROMACS 正常工作，可以从Max Planck Institute for Multidisciplinary Sciences （**[**https://www.mpinat.mpg.de/grubmueller/bench**](https://www.mpinat.mpg.de/grubmueller/bench)**）提供的GROMACS基准测试集中下载TPR文件来进行性能测试。这些基准测试集包含了不同系统大小的模拟系统，从6千到1200万原子不等，非常适合用来评估GROMACS的性能。比如下载：**[**benchMEM (82k atoms, protein in membrane surrounded by water, 2 fs time step)**](https://www.mpinat.mpg.de/benchMEM) 这个 tpr 文件进行测试 `mdrun`，本次笔者使用此文件进行测试。
- **如果网络慢，可本地下载解压后，用 `scp` 上传到服务器**
- Run test cases to ensure GROMACS works properly, From the Max Planck Institute for Multidisciplinary Sciences ([https://www.mpinat.mpg.de/grubmueller/bench)](https://www.mpinat.mpg.de/grubmueller/bench)) provides GROMACS benchmark test set download TPR files for performance test. These benchmark sets, which contain simulations of different system sizes ranging from 6 to 12 million atoms, are ideal for evaluating GROMACS performance. For example, download: benchMEM (82k atoms, protein in membrane surrounded by water, 2 fs time step) this tpr file to test `mdrun`
- If the network is slow, you can download and decompress it locally and upload it to the server using `scp`
```
# mpi版Gromacs用mpirun，对于root用户，OpenMPI要求每次执行mpirun命令都得带着-allow-run-as-root选项才行，解决方法参考：http://sobereva.com/409
gmx2024 mpirun -allow-run-as-root -np 36 gmx_mpi mdrun -s benchMEM.tpr -noconfout -nsteps 10000 -pin on -nb gpu
# 普通单精度Gromacs，使用1颗cpu
gmx2024 mdrun -nt 36 -s benchMEM.tpr -noconfout -nsteps 10000 -pin on -nb gpu # 笔者单次测试结果68.346 ns/d
gmx2024 mdrun -ntmpi 4 -ntomp 9 -s benchMEM.tpr -noconfout -nsteps 10000 -pin on -nb gpu # 笔者单次测试结果53.701 ns/d
```
