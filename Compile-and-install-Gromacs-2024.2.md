# 准备工作--Preparatory work
- **声明**：本编译过程使用系统为 centos7，故而对所用编译环境检查是否升级，准备工作部分需要根据自己实际系统环境及其自带的 gcc、cmake 等版本进行操作。
- **Declaration**: This compilation process uses the centos7 system, so check whether the compilation environment used is upgraded, and the preparation part needs to operate according to their actual system environment and its own gcc, cmake and other versions.
## 检查系统编译环境-Check the system compilation environment
```
mpirun --version
cmake3 --version 
gcc --version 
nvcc --version 
fftw-wisdom --version #It can be installed by Gromacs itself
```
## 升级 GCC-Upgrade GCC
```
# 经检查cuda版本为11，为了兼容只能升级到gcc 9-cuda version is checked to be 11, and can only be upgraded to gcc 9 for compatibility
sudo yum -y install centos-release-scl 
sudo yum install devtoolset-9-gcc devtoolset-9-gcc-c++ devtoolset-9-binutils
scl enable devtoolset-9 bash  #Enable devtoolset-9 with the scl command, which sets environment variables in the current session to use the new version of GCC
echo "source /opt/rh/devtoolset-9/enable" >> ~/.bashrc
gcc --version
```
## 升级 CMake-Upgrade CMake
```
sudo yum install openssl-devel
sudo yum remove cmake3  # Uninstall the current cmake3
wget https://cmake.org/files/v3.29/cmake-3.29.6.tar.gz
tar -zxvf cmake-3.29.6.tar.gz
cd cmake-3.29.6
./bootstrap  #./bootstrap 是一个自动化脚本，设置编译环境，使能够编译并安装CMake-bootstrap is an automation script that sets up a compilation environment that enables CMake to be compiled and installed
make 
sudo make install
```
## 安装 OpenMPI 库-Install the OpenMPI library
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
ompi_info
```
## （安装**快速傅立叶变换库**FFTW-Install FFTW）
```
sudo yum remove fftw 
wget http://www.fftw.org/fftw-3.3.10.tar.gz 
tar -xvf fftw-3.3.10.tar.gz 
cd fftw-3.3.10 
./configure --enable-float --prefix=/usr/local/fftw 
make 
sudo make install
echo "export LD_LIBRARY_PATH=/usr/local/fftw/lib:\$LD_LIBRARY_PATH" >> ~/.bashrc  # 如果安装到了非标准路径，更新环境变量，将 FFTW 的库路径添加到 LD_LIBRARY_PATH
source ~/.bashrc
```
# 下载 Gromacs 源代码-Download the Gromacs source code

- 提前检查系统信息，如 cuda 版本、gpu 驱动版本 、SIMD 等
- Check system information in advance, such as the cuda version, gpu driver version, and SIMD
```
lscpu  # check CPU information, such as SIMD
nvidia-smi # check nvidia driver and cuda
```
```
#最好提前 mkdir 到特定目录，比如: 用户名-- GMX--GMX2024.2
wget https://ftp.gromacs.org/gromacs/gromacs-2024.2.tar.gz
wget https://ftp.gromacs.org/regressiontests/regressiontests-2024.2.tar.gz
tar -xvzf gromacs-2024.2.tar.gz
tar -xvzf regressiontests-2024.2.tar.gz
```
# Gromacs 的编译与安装-Compile and install Gromacs

- 若有多个版本的 gromacs 则需要分开安装
- If there are multiple versions of gromacs, they need to be installed separately
```
mkdir build #创建一个构建目录 build 并运行 CMake 来配置编译选项。
cd build
cmake ../gromacs-2024.2 \ 
  -DGMX_BUILD_OWN_FFTW=ON \
  -DREGRESSIONTEST_DOWNLOAD=OFF \
  -DREGRESSIONTEST_PATH=../regressiontests-2024.2 \
  -DCMAKE_C_COMPILER=gcc \  
  -DGMX_SIMD=AVX2_256 \ 
  -DGMX_MPI=ON \
  -DGMX_GPU=CUDA \  
  -DCMAKE_INSTALL_PREFIX=~/Software/GMX/2024.2-cuda  # 指定安装路径，可自行修改
```
```
make -j 36 
make check 
make install 
```
# 配置环境变量-Configuring environment variables

- 若有多个版本的 gmx 还需要改名以便调用特定版本的 gmx
- If you have more than one version of gmx, you also need to change the name in order to call a specific version of gmx
```
echo -e 'export PATH=/jer/GMX2024.2/build/bin:$PATH\nalias gmx2024="gmx"' >> ~/.bashrc && source ~/.bashrc
# 其中/jer/GMX2024.2/build/bin应改为自己实际路径
gmx2024 --version 
```
# 性能测试-Performance test

- 运行测试案例，确保 GROMACS 正常工作，可以从Max Planck Institute for Multidisciplinary Sciences （[https://www.mpinat.mpg.de/grubmueller/bench](https://www.mpinat.mpg.de/grubmueller/bench)）提供的GROMACS基准测试集中下载TPR文件来进行性能测试。这些基准测试集包含了不同系统大小的模拟系统，从6千到1200万原子不等，非常适合用来评估GROMACS的性能。比如下载：[benchMEM (82k atoms, protein in membrane surrounded by water, 2 fs time step)](https://www.mpinat.mpg.de/benchMEM) 这个 tpr 文件进行测试 `mdrun`
- 如果网络慢，可本地下载解压后，用 `scp` 上传到服务器
- Run test cases to ensure GROMACS works properly, From the Max Planck Institute for Multidisciplinary Sciences ([https://www.mpinat.mpg.de/grubmueller/bench)](https://www.mpinat.mpg.de/grubmueller/bench)) provides GROMACS benchmark test set download TPR files for performance test. These benchmark sets, which contain simulations of different system sizes ranging from 6 to 12 million atoms, are ideal for evaluating GROMACS performance. For example, download: benchMEM (82k atoms, protein in membrane surrounded by water, 2 fs time step) this tpr file to test `mdrun`
- If the network is slow, you can download and decompress it locally and upload it to the server using `scp`
```
gmx2024 mpirun -allow-run-as-root -np 36 gmx_mpi mdrun -s benchMEM.tpr -noconfout -nsteps 10000 -pin on -nb gpu #对于root用户，OpenMPI要求每次执行mpirun命令都得带着-allow-run-as-root选项才行，解决方法参考：http://sobereva.com/409
```
