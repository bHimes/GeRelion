# GeRelion

GeRelion is a GPU-enhanced parallel implementation of single particle cryo-EM image processing.
Developed by Huayou Su, Wen Wen, Xiaoli Du, Xicheng Lu,Dongsheng Li from National University of Defense Technology and Maofu Liao from Harvard Medical School. 
For the current version, we implemented 
the most time-comsuming components of RELION with GPU, including _3D classificaion_ and _auto-refine_.

The implementation is built upon RELION (REgularised LIkelihood Optimisation).
RELION is a stand-alone computer program that employs an empirical Bayesian approach to refinement of (multiple) 3D reconstructions or 2D class averages in electron cryo-microscopy (cryo-EM). It is developed in the group of Sjors Scheres at the MRC Laboratory of Molecular Biology.
[More detailed information about RELION can be found](http://www2.mrc-lmb.cam.ac.uk/relion/index.php/Main_Page)

The original RELION codes for data read/write and MPI communication are unmodified, 
and the flow control of progressive processing in the original RELION is kept. 
The _*.cu_ source files  and their corresponding header files in directory _src_ are developed by GeRelion team.
The _*.cpp_ source files  and their corresponding header files in directory _src_ are from relion-1.3.
Some of the _*.ccp_ and _*.hpp_ files from relion are modified by GeRelion team.

We integrated a new algorithm into GeRelion, named dynamic singal subtraction (DSS). The parameters required for DSS are --red_mask and --yellow_map (or --yellow_mask).

--red_mask (red_mask_file): User-provided mask for the wanted region. During DSS, GeRelion multiplies the 3D reconstruction from the last iteration with this mask to generate the reference map of the wanted region.  

--yellow_mask (yellow_mask_file): User-provided mask for the unwanted region. During DSS, GeRelion multiplies the 3D reconstruction from the last iteration with this mask to generate the reference map of the unwanted region, i.e. �yellow map�. This yellow map is used to generate 2D projections, which are then subtracted from the original particle images. 

--yellow_map (yellow_map_file): User-provided �yellow map�, which will be used in every iteration. When --yellow map is present, --yellow_mask is ignored. (The unwanted region often becomes weaker during iterative refinement, and thus contains not enough signal for subtraction. Therefore, it is often preferred to use a fixed �yellow_map� instead of a �yellow_mask� to generate �yellow_map� during iterations.)

--extra_iter (0): Extra number of iterations to perform. GeRelion auto-refine iterates additional n cycles even after the convergence is achieved.

--fsc_mask (fsc_mask_file): User-provided mask to be applied on the half volumes before FSC calculation.

## Prerequisites
----------------
1. Hardware requirements
To use the program GeRelion, you should have the CUDA-enabled GPUs from NVIDIA Corporation.
GeRelion prefers NVIDIA GPUs with arch_sm equals 3.5 or prior.
2. Software requirements
The CUDA Toolkit must be installed successfully. In addition,
the thrust library and [MPI](https://www.open-mpi.org/) pragraming interface should be installed. 
3. The thiry party library fftw and fltk is required, both of them are put int the folder _external_.
 
## Compiling the GeRelion
-------------------------
1. Change the directory of your project in Makefile, check line 1.
   CUDA_DIR := path to your cuda directory installed (such as /usr/local/cuda-7.5) 

2. Move to your GeRelion project folder and run the script _INSTALL.sh_.
The compilation will last for several minites due to the usage of thrust library. 
The executable binary file will be put in  _bin_ directory of the project. 

## Edit the environment file 
-------------------

Replace ��$GERELION_HOME�� with the actual path of your GeRelion directory.
If csh or tcsh, add the following lines to your ~/.cshrc. 

export LD_LIBRARY_PATH=$GERELION_HOME/lib:$LD_LIBRARY_PATH
export PATH=$GERELION_HOME/bin:$PATH

If bash, add the following lines to your ~/.bashrc:

setenv LD_LIBRARY_PATH $GERELION_HOME/lib:$LD_LIBRARY_PATH
setenv PATH $GERELION_HOME/bin:$PATH


## Running GeRelion
-------------------

The current version of GeRelion only supports job submission via command line (GUI will be implemented soon).
GeRelion uses the same parameters as RELION, so you can use RELION GUI 
"print command" to get all the necessary parameters. You can also skip 
GUI, and use [samrelion.py](http://liao.hms.harvard.edu/node/20) to conveniently print out all parameters. 
Note that, without using RELION pipeline, [samrelion.py](http://liao.hms.harvard.edu/node/20) can generate 
initial star files, analyze and modify data star file, generate the star 
files with particular class numbers, merge small groups, etc.

One GeRelion-specific parameter (--mode) is used to indicate running mode of GeRelion: 0 for CPU, and 1 for GPU. 

A step-by-step instruction to test GeRelion using the TRPV1 data set:

1. Download the TRPV1 particle stack [tv1_f01-30.mrc]( https://www.ebi.ac.uk/pdbe/emdb/empiar/entry/10005/), and rename it as ��tv1.mrcs��.
2. Download the TRPV1 3D reconstruction [emd_5778.map](http://emsearch.rutgers.edu/atlas/5778_downloads.html), and rename it as ��tv1.mrc��
3. Download this star file [tv1.star](https://1drv.ms/f/s!AnzI0m5_no6OgTo_1Mi-NFKgnZTm).
4. The following commands assume that you have 2 GPU nodes (��node01�� and ��node02��), each containing 4 GPU cards. ��$GERELION_HOME�� should be replaced by the actual path of your GeRelion directory. 

__Run the following command for 3D auto-refine (with C4 symmetry):__  
mpirun --np 9 --host node01,node02 $GERELION_HOME/bin/gereline_refine_mpi 
--o REF01 --auto_refine --split_random_halves --i tv1.star --particle_diameter 200 --angpix 1.2156 --ref tv1.mrc --firstiter_cc --ini_high 60 --ctf --ctf_corrected_ref --flatten_solvent --zero_mask --oversampling 1 --healpix_order 2 --auto_local_healpix_order 4 --offset_range 5 --offset_step 2 --sym C4 --low_resol_join_halves 40 --norm --scale  --j 1 --memory_per_thread 4 --dont_combine_weights_via_disc --mode 1 

__Run the following command for 3D classification (3 classes, without symmetry):__  
mpirun --np 9 --host node01,node02 $GERELION_HOME/bin/gereline_refine_mpi 
--o CLS01 --i tv1.star --particle_diameter 200 --angpix 1.2156 --ref tv1.mrc --firstiter_cc --ini_high 60 --ctf --ctf_corrected_ref --iter 25 --tau2_fudge 4 --K 3 --flatten_solvent --zero_mask --oversampling 1 --healpix_order 2 --offset_range 5 --offset_step 2 --sym C1 --norm --scale  --j 1 --memory_per_thread 4 --dont_combine_weights_via_disc --mode 1

__If you only have one node with 4 GPU cards, the commands are:__  
mpirun --np 5 $GERELION_HOME/bin/gereline_refine_mpi ... ...

## License
----------
GeRelion is released under the terms of the GNU [General Public License](https://opensource.org/licenses/gpl-license) as published by the Free Software Foundation.


Contact:
Huayou Su
huayousu@163.com
