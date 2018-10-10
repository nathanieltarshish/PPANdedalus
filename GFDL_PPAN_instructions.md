Install notes for GFDL PPAN Cluster 
====================================


This installation uses conda to install BLAS, openmpi, and the various Python packages required by dedalus. The only exceptions are HDF5, FFTW, and h5py which are built manually from source to ensure that parallelization is enabled. 

For these manual installations, the source download must be conducted on the `public` nodes, which can access the internet. **Installation must be done on the analysis nodes**, which have the proper compiliers. 

By default, these instructions create the directory ``/nbhome/${USER}/software`` and install dedalus, HDF5, FFTW, and h5py within this directory. We assume the user is running the default c-shell. 

Download Source Files to Public 
-------------------------------

These instructions also assume you have installed Anaconda (or Miniconda) to ``/nbhome/${USER}``. To start, let's create a new conda environemnt for your dedalus installation. 

Login into ``public`` and create a  ``dedalus.yml`` file with the following contents:

```
name: dedalus 
channels:
  - defaults
  - conda-forge 
dependencies:
  - python=3
  - numpy
  - nomkl
  - cython 
  - scipy 
  - mpi4py=3.0 
  - openblas 
  - openmpi 
  - matplotlib
  - pathlib
  - docopt
  - pkgconfig
```

Setup the dedalus environment, 
```
conda env create -f dedalus.yml
```

We will now create a file that sets the environment variables necessary for subsequently building the packages and activates the conda environment.

Create a file entitled ``dedalus_paths.csh`` with the following content:
```
#DEDALUS SETUP

module load gcc

setenv CC mpicc

source /nbhome/${USER}/anaconda3/etc/profile.d/conda.csh
conda activate dedalus

setenv BLAS /nbhome/${USER}/anaconda3/envs/dedalus/lib/libopenblas.so
setenv MPI_PATH /nbhome/${USER}/anaconda3/envs/dedalus/lib/openmpi
setenv LD_LIBRARY_PATH ${MPI_PATH}:${BLAS}:${LD_LIBRARY_PATH}

setenv HDF5_DIR /nbhome/${USER}/software/hdf5
setenv HDF5_VERSION 1.10.1
setenv HDF5_MPI "ON"
setenv PATH ${HDF5_DIR}/bin:${PATH}
setenv LD_LIBRARY_PATH ${HDF5_DIR}/lib:${LD_LIBRARY_PATH}

setenv FFTW_PATH /nbhome/${USER}/software/fftw
setenv FFTW_VERSION 3.3.6-pl2
setenv PATH ${FFTW_PATH}/bin:${PATH}
setenv LD_LIBRARY_PATH ${FFTW_PATH}/lib:${LD_LIBRARY_PATH}

setenv DEDALUS_REPO /nbhome/${USER}/software/dedalus

setenv H5PY_REPO /nbhome/${USER}/software/h5py

#add or append dedalus to python path 
if (! $?PYTHONPATH) then
  setenv PYTHONPATH "${DEDALUS_REPO}"
else
  setenv PYTHONPATH "${DEDALUS_REPO}":${PYTHONPATH}
endif
```

Now run, ``source dedalus_paths.csh`` to set these environmental paths. 
 
We are now ready to create the directories and download the source files. To do this, run the following script:
 
```
# download HDF5 from source
mkdir -p ${HDF5_DIR}
cd ${HDF5_DIR}
wget https://support.hdfgroup.org/ftp/HDF5/current/src/hdf5-${HDF5_VERSION}.tar

# download FFTW built from source
mkdir -p ${FFTW_PATH}
cd ${FFTW_PATH}
wget http://www.fftw.org/fftw-${FFTW_VERSION}.tar.gz

# Dedalus from mercurial
hg clone https://bitbucket.org/dedalus-project/dedalus ${DEDALUS_REPO}
cd ${DEDALUS_REPO}

# download h5py from source
cd /nbhome/${USER}/software/
git clone https://github.com/h5py/h5py.git
```

Build and Install on Analysis
------------------------
Login into the analysis cluster and  ``source dedalus_paths.csh``. To build the packages, run the following script 

```
# build HDF5 from source
cd ${HDF5_DIR}
tar -xvf hdf5-${HDF5_VERSION}.tar
cd hdf5-${HDF5_VERSION}
./configure --prefix=${HDF5_DIR} \
CC=mpicc \
CXX=mpicxx \
F77=mpif90 \
MPICC=mpicc \
MPICXX=mpicxx \
--enable-shared \
--enable-parallel
make -j4
make install

# install h5py and link hdf5  
cd ${H5PY_REPO}
python setup.py configure --mpi --hdf5=${HDF5_DIR}
python setup.py build
python setup.py install

# build HDF5 from source
cd ${FFTW_PATH}
tar -xvzf fftw-${FFTW_VERSION}.tar.gz
cd fftw-${FFTW_VERSION}
./configure --prefix=${FFTW_PATH} \
CC=mpicc \
CXX=mpicxx \
F77=mpif90 \
MPICC=mpicc \
MPICXX=mpicxx \
--enable-shared \
--enable-mpi \
--enable-openmp \
--enable-threads
make -j4
make install

# install Dedalus 
cd ${DEDALUS_REPO}
python setup.py build_ext --inplace
```

Notes
-----
Based on the [MIT Engage cluster install notes](http://dedalus-project.readthedocs.io/en/latest/machines/engaging/engaging.html). Written by [Spencer Clark](https://github.com/spencerkclark) and Nathaniel Tarshish on June 21, 2018. We've had FFTW issues running with the default version of numpy (1.14) selected by conda. Therefore, we have pinned numpy to 1.12 in the `.yml` file. We anticipate that setup on `gaea` could be done in a similar fashion. As a result of mercurial not being available on analysis, we have opted for the setting up the requirements through conda and excluding `hgapi`. 
