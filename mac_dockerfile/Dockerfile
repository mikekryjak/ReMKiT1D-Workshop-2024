FROM --platform=arm64 ubuntu:latest
LABEL maintainer="stefan.mijin@ukaea.uk"
LABEL version="1.0.0"
LABEL description="Docker container for ReMKiT1D CI"

# Disable Prompt During Packages Installation

ARG DEBIAN_FRONTEND=noninteractive

RUN apt update && apt install software-properties-common -y

RUN add-apt-repository -y ppa:ubuntu-toolchain-r/test

# Update and Installs Required Packages for ReMKiT1D and other Packages

RUN apt update \
        && apt install -y \
        gfortran-11 \
        g++-11 \
        python3 \
        python3-dev \
        pip \
        libblas-dev \
        liblapack-dev \
        m4 \
        wget \
        git \
        gcovr \ 
        gcc-arm-linux-gnueabihf \
        build-essential \
        pkg-config

# Packages need to be Compiled and Installed Using Fortran-11

# CMake Commands State the Required Compilers to use
ENV FC=gfortran-11
ENV CC=gcc-11
ENV CXX=g++-11

WORKDIR /home

# ReMKiT1D Requires CMake-3.18.0 and pFUnit-4.2.2 has some Incompatibilities with Newer Versions of CMake (v3.25.0-rc1)
RUN wget https://github.com/Kitware/CMake/releases/download/v3.18.0/cmake-3.18.0.tar.gz

RUN tar -zvxf cmake-3.18.0.tar.gz

WORKDIR /home/cmake-3.18.0

RUN ./bootstrap -- -DCMAKE_USE_OPENSSL=OFF

RUN make

RUN make install

WORKDIR /home

# ReMKiT1D Requires MPI and mpich-3.4.2 has been Tested and is used in the main.yml Workflow Script
RUN wget -O mpich.tar.gz https://www.mpich.org/static/downloads/3.4.2/mpich-3.4.2.tar.gz

RUN tar xfz mpich.tar.gz

RUN mkdir mpich-install

WORKDIR /home/mpich-3.4.2

# ReMKiT1D Requires gfortran-11, gcc-11 and g++-11. CMake Installs the Package to the mpich-install Folder
RUN ./configure CC=gcc-11 CXX=g++-11 FC=gfortran-11 --prefix=/home/mpich-install  FFLAGS=-fallow-argument-mismatch --with-device=ch3

RUN make

RUN make install

WORKDIR /home

# Include the mpich-install PATH for Compilation of ReMKiT1D
ENV PATH=/home/mpich-install/bin:$PATH

# PETSc-3.17 is the Newest Release that has been Tested on ReMKiT1D (3.18 not Compatible)
RUN git clone -b release-3.17 https://gitlab.com/petsc/petsc.git petsc

WORKDIR petsc/

#PETSc Requires MPI and PETSc is Told Where MPI (mpich-4.2.2) is Installed
RUN ./configure --with-mpi-dir=/home/mpich-install --download-hypre=1 --download-fblaslapack=1 --with-debugging=0 COPTFLAGS=-O3 CXXOPTFLAGS=-O3 FOPTFLAGS=-O3

RUN make PETSC_DIR=/home/petsc PETSC_ARCH=arch-linux-c-opt all check

WORKDIR /home

# pFUnit-4.2.2 is Installed for the use of Unit Testing
RUN git clone -b v4.2.2 https://github.com/Goddard-Fortran-Ecosystem/pFUnit.git

WORKDIR pFUnit

RUN mkdir build

WORKDIR build

# ReMKiT1D Requires gfortran-11, gcc-11 and g++-11. CMake Installs the Package to the 'installs' Folder
RUN cmake .. -DSKIP_OPENMP=yes -DCMAKE_Fortran_COMPILER=gfortran-11 -DCMAKE_C_COMPILER=gcc-11 -DCMAKE_CXX_COMPILER=g++-11 -DCMAKE_INSTALL_PREFIX=/home/installs/pFUnit

RUN make install

WORKDIR /home

# ReMKiT1D is Currently using json-fortran-8.2.5
RUN git clone -b 8.2.5 https://github.com/jacobwilliams/json-fortran.git

WORKDIR json-fortran

RUN mkdir build

WORKDIR build

# ReMKiT1D Requires gfortran-11, gcc-11 and g++-11. CMake Installs the Package to the 'installs' Folder
RUN cmake .. -DCMAKE_Fortran_COMPILER=gfortran-11 -DCMAKE_INSTALL_PREFIX=/home/installs/json-fortran -DSKIP_DOC_GEN=TRUE

RUN make install

WORKDIR /home

# HDF5 is Required and HDF5-1.13.0 is used within the main.yml Workflow Script
RUN git clone -b hdf5-1_13_0 https://github.com/HDFGroup/hdf5.git

WORKDIR hdf5

RUN mkdir build

WORKDIR build

# ReMKiT1D Requires gfortran-11, gcc-11 and g++-11. CMake Installs the Package to the 'installs' Folder
RUN cmake .. -DCMAKE_Fortran_COMPILER=gfortran-11 -DCMAKE_C_COMPILER=gcc-11 -DCMAKE_CXX_COMPILER=g++-11 -DHDF5_BUILD_FORTRAN=ON -DCMAKE_INSTALL_PREFIX=/home/installs/hdf5

RUN make install

WORKDIR /home

RUN echo LD_LIBRARY_PATH=/home/mpich-install/lib:$LD_LIBRARY_PATH

# Add the PATH to assist CMake and Make Find the Required Compilation Files
ENV PFUNIT_DIR=/home/installs/pFUnit
ENV PETSC_DIR=/home/petsc
ENV PETSC_ARCH=arch-linux-c-opt
ENV PATH=$PATH:/home/installs/
ENV PATH=$PATH:/home/installs/petsc
ENV PATH=$PATH:/home/installs/hdf5
ENV PATH=$PATH:/home/installs/json-fortran/jsonfortran-gnu-8.2.5

# Install the Python library
RUN git clone -b dev-v1.1.0 https://github.com/ukaea/ReMKiT1D-Python.git

RUN pip install ./ReMKiT1D-Python/

WORKDIR /home 

# Install ReMKiT1D
RUN git clone -b dev-v1.1.0 https://github.com/ukaea/ReMKiT1D.git

WORKDIR /home/ReMKiT1D 

RUN mkdir debug

WORKDIR /home/ReMKiT1D/debug

RUN which python3

RUN cmake .. \
-DPYTHON_INCLUDE_DIR=$(python3 -c "import sysconfig; print(sysconfig.get_path('include'))")  \
-DPYTHON_LIBRARY=$(python3 -c "import sysconfig; print(sysconfig.get_config_var('LIBDIR'))") \
-DPYTHON_EXECUTABLE:FILEPATH=`which python3`

RUN make

# Test ReMKiT1D 
RUN make test > /home/ReMKiT1D_debug_test.out

WORKDIR /home/ReMKiT1D 

RUN mkdir build 

WORKDIR /home/ReMKiT1D/build

RUN cmake .. \
-DPYTHON_INCLUDE_DIR=$(python3 -c "import sysconfig; print(sysconfig.get_path('include'))")  \
-DPYTHON_LIBRARY=$(python3 -c "import sysconfig; print(sysconfig.get_config_var('LIBDIR'))") \
-DPYTHON_EXECUTABLE:FILEPATH=`which python3`

RUN make

# Test ReMKiT1D 
RUN make test > /home/ReMKiT1D_build_test.out

WORKDIR /home

# Install other required Python packages and test RMK_Support

RUN pip install ipywidgets ipykernel jupyter_bokeh pytest 

RUN pytest ./ReMKiT1D-Python/RMK_support/ > /home/RMK_support_test.out

RUN git clone https://github.com/ukaea/ReMKiT1D-Workshop-2024