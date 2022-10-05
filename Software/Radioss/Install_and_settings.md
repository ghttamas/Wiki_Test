# Install and settings and run

## Install
### Install tools
```bash
apt-get update
apt-get upgrade
apt-get install build-essential
apt-get install gfortran
apt-get install cmake
apt-get install perl
apt-get install git-lfs
```

### Build OpenMPI
```bash
mkdir build_openmpi
cd build_openmpi
wget https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-4.1.2.tar.gz
tar -xvzf openmpi-4.1.2.tar.gz
cd openmpi-4.1.2
./configure --prefix=/opt/openmpi/4.1.2
make
make install
```

### Build OpenRadioss
```bash
git lfs install
mkdir build_openradioss
cd build_openradioss
mkdir release_date (ex. 20220924)
cd release_date 
# Git acces must be setup before the next step
git clone git@github.com:OpenRadioss/OpenRadioss.git
# Building starter
cd OpenRadioss/starter
./build_script.sh -arch=linux64_gf
cd ../engine
# Building smp executable
./build_script.sh -arch=linux64_gf
# Building executable with openmpi
./build_script.sh -arch=linux64_gf -mpi=ompi -mpi-root=/opt/openmpi/4.1.2
# Building tools
cd ../tools/anim_to_vtk/linux64
./build.bash
cd ../../th_to_csv/linux64
./build.bash
```

## Profile file

```bash
# the amount of stacksize memory
ulimit –s unlimited
export OPENRADIOSS_PATH=[Path to OpenRadioss root directory]
export RAD_CFG_PATH=$OPENRADIOSS_PATH/hm_cfg_files
export LD_LIBRARY_PATH=$OPENRADIOSS_PATH/extlib/hm_reader/linux64/:$OPENRADIOSS_PATH/extlib/h3d/lib/linux64/:$LD_LIBRARY_PATH
export OMP_STACKSIZE=512m
# number of OpenMP threads
export OMP_NUM_THREADS=[N]
# with OpenMPI
export LD_LIBRARY_PATH=/opt/openmpi/lib:$LD_LIBRARY_PATH
export PATH=/opt/openmpi/bin:$PATH
export OMP_NUM_THREADS=[N]
```

## Run
There are two parts of the Radioss simulation, the Starter and the Engine.
The Starter is an input data checker and must successfully complete without errors before the simulation can be started with the Engine.
The Radioss Starter is responsible for checking the model consistency and reporting any errors or warnings in the output file. If there are no errors in the model, the Radioss Starter creates initial restart file(s), `runname_0000_CPU#.rst`. There is one restart file created for each SPMD MPI domain requested for the solution.
The Radioss Engine takes as input the Radioss Engine file, `runname_0001.rad` and the initial restart file(s) created by the Radioss Starter. The Radioss Engine files describes the solution control and output for the simulation. While the Radioss Engine is running, an Engine output file, `runname_0001.out`, is created which contains statistics about the simulation including time, time step, current system energies, energy error, and mass error.

### Run Starter
`starter_linux64_gf -i Example_0000.rad`
#### Starter's commad line options
 - **-help	(-h)**	Print help message
- **-version	(-v)**	Print Radioss release information
- **-input FILE	(-i)**	Set Radioss Starter input file
- **-nspmd INTEGER	(-np)**	Set number of SPMD domains
- **-nthread INTEGER	(-nt)**	Set number of threads per SPMD domains
- **-notrap**	Disable error trapping
- **-check**	Option to check the model. Disable domain decomposition and restart file writing
- **-outfile=output path**	Defines the output file directory for all files created
- **-HSTP_WRITE**	Write an `<root_name>_0000.rad2hst` file containing model the `/PARAMETER` information.
- **-HSTP_READ**	Read in a `hst_input.hstp` file and replace the input deck’s `/PARAMETER` with the ones defined in the `.hstp` file
- **-rxalea REAL**	Activation of nodal random noise with a value xalea
- **-rseed REAL**	Optional value: Set the seed value for nodal random noise.
- **-dylib FILE**	Set name of the dynamic library for Radioss User Subroutines.
- **-mds_libpath PATH** Set path to the dynamic library for Multiscale Designer

### Starter script

```bash
#!/bin/bash
starter_linux64_gf -i {STARTER_INPUT}_0000.rad -np 1
EXIT_CODE=$?
echo ${EXIT_CODE}
```

### Run Engine
### Engine starter
#### SMP mode
`engine_linux64_gf -i {STARTER_INPUT}_0001.rad -nt 4`
#### MPI mode
`mpirun [mpi options] engine_linux64_gf_ompi [engine options]`
`mpirun -np 4 engine_linux64_gf_ompi -nt 2 -i Example_0001.rad`
The `-nspmd` value (or the Nspmd field of `/SPMD` starter input card) must match the `mpirun -np` value!
```bash
starter_linux64_gf -nspmd 4 -input Example_0000.rad
mpirun -np 4 engine_linux64_gf_ompi -input Example_0001.rad

export OMP_NUM_THREADS=[N]
starter_linux64_gf -i Example2_0000.rad -np [P]
mpiexec -n [P] engine_linux64_gf_ompi -i Example2_0001.rad
```

#### Engine's commad line options
- **-help	(-h)**	Print help message
- **-version	(-v)**	Print Radioss release information
- **-input FILE	(-i)**	Set Radioss Engine input file
- **-nthread INTEGER	(-nt)**	Set Number of SMP threads per SPMD domain
- **-notrap**	Disable error trapping
- **-norst**	Disable restart `*.rst` file writing during and at the end of the computation
- **-outfile=output path**	Defines the output file directory for all files created
- **-dylib FILE**	Set name of the dynamic library for Radioss User Subroutines
- **-mds_libpath PATH**	Set path to the dynamic library for Multiscale Designer

## Control file (C-File)
The optional control file is used to get information about a currently running analysis. It has the same prefix name as the current Engine input file but ends in `*.ctl`. If the Engine file is `Example_0001.rad` then `Example_0001.ctl`.
The control file can be created using a text editor and saved in the directory where Radioss is writing the Engine output file. The following options can be used:
- **/ANIM** Create an extra animation file (Axxx)
- **/CHKPT** Create a file named CHECK_DATA which contains `/RERUN`(rerun_engine_r.htm "Engine Keyword Permits to continue a previous Radioss run.") commands to continue a simulation if it is stopped. Usually used in combination with `/STOP` (stop_engine_r.htm "Engine Keyword The Engine is stopped, due to reached energy error ratio criteria, total mass ratio criteria or nodal mass ratio criteria.") to stop the simulation. Not available with the implicit solution
- **/CYCLE/No_of_cycle** The control file commands will be executed at the specified cycle number
- **/H3D** Write animation data to the *.h3d file
- **/INFO** Returns information on current cycle, current global energies, current time step. This information is always written for all options
- **/KILL** Kill the simulation and do not create a restart file
- **/RFILE** Create a restart file
- **/STOP** Stop the simulation and create a restart file
- **/TIME/timeValue** The other control file commands will be executed at the specified simulation time
Example:
```
/TIME/.1
/ANIM
/CHKPT
/STOP
```
