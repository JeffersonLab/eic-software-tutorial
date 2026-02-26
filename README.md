
> **Warning**
>
> This is an unofficial documentation. An experiment to create easy TL;DR; documentation on EIC EPIC software for JLab users. Use it on your own risk. Check dates if it is outdated or not. 

# Run simulation and reconstruction 

This instruction goes through these steps: 

1. Run eic-shell - EIC software package
2. Convert MCEG from LUND or PYTHIA6 to HepMC3
3. Apply crossing angle and beam effects to MCEG 
4. Run MCEG through the full simulation
5. Run full simulation results through the reconstruction


## Quick start tutorial

```bash
# == 0 == Install eic-shell
# https://eic.github.io/tutorial-setting-up-environment/setup.html
curl --location https://get.epic-eic.org | bash

# == 1 == Assuming eic-shell is installed
# on ifarm: 
./eic-shell

# == 2 == Convert MCEG to HepMC3

# Install mcconv (in theory this should be onlhy once)
pip3 install mcconv

# Make sure ~/.local/bin is in the path
export PATH=/home/$USER/.local/bin:$PATH

# mcconv should work now
mcconv --help

# Convert GEMC LUND (HallB) file to HepMC3
# (!) Important to add beam particles energies for LUND format
mcconv -o headon.hepmc -f lund_gemc -b 10x100 dvcs_lund_51520.txt

# == 3 == Add crossing angle and beam effects

# Add CA and BE using the Afterburner
abconv headon.hepmc -o sim_input

# As the result two file will be creatred
#   - sim_input.hepmc      - HepMC data with crossing angle, 
#   - sim_input.hist.root  - validation histos


# == 4 == Run DD4Hep full simulation

# Detectors live in
# /opt/detectors
# one can select particular configuration as
# source /opt/detector/epic-xxx/bin/thisepic.sh
#
# or one can set the default detector (now points to epic-nightly)
source /opt/detector/epic-main/bin/thisepic.sh

# Run simulation for 1000 events
# (!) Using *.edm4hep.root in the output is mandatory
ddsim --compactFile=$DETECTOR_PATH/epic.xml -N=1000 --outputFile=sim_output.edm4hep.root --inputFiles sim_input.hepmc

# == 5 == Run EICrecon reconstruction
# (!) Use the same geometry/detector as for simulation
#     e.g. source /opt/detector/setup.sh
eicrecon -Ppodio:output_file=processed.edm4eic.root -Phistsfile=processed.hist.root sim_output.edm4hep.root

```

## 1. Using `eic-shell`

The EIC environment `eic-shell` is a singularity/docker container with a 
curated selection of software components. It is the recommended way to use
`eicrecon` as it already has all of the dependencies compiled with matching
version numbers. This requires either singularity or docker to be installed
on your local machine. Below are some quick start instructions for using
Singularity. Docker/podman containers also could be used. 

Singularity is installed on JLab CUE but for local machines [use the instruction in the end of this document](#How-to-install-Singularity)

**Quick start for eic-shell**

```bash
mkdir ~/eic
cd ~/eic

curl --location https://get.epic-eic.org | bash
./eic-shell

# or, if /cvmfs is available (on ifarm):

singularity exec /cvmfs/singularity.opensciencegrid.org/eicweb/eic_xl:nightly eic-shell

#or if docker or podman are available
docker run --rm -it -v <host/data/dir>:/mnt/data eicweb/eic_xl:nightly eic-shell
```

If you run it on the JLab CUE you should first enable singularity like 

```bash
module load singularity/3.10.0
```

<details>
  <summary>[spoiler] Singularity versions at JLab</summary>
  <p>
    To check what singularity versions are available with module: 

    ```bash
    module avail
    ```
  </p>
</details>
<br/>

> Full tutorials on setting up you environment with `eic-shell` can be found here:
> - [EIC Tutorial: Setting Up Your Environment](https://eic.github.io/tutorial-setting-up-environment/index.html)
> - [`eic-shell` video tutorial](https://www.youtube.com/watch?v=Y0Mg24XLomY)


<br/>

## 2 Convert MCEG to HepMC3

Simulation software strictly accepts only HepMC3 as an input format. (though HepMC2 should work too)

Currently there are 2 packages that allow to convert different MCEG to HepMC

- [EicSmear](https://github.com/eic/eic-smear) - and old (means well tested but cumbersome to use) and reliable package  that allows to convert various (and especially old) MCEG to HepMC format. 
   [Instruction how to convert to HepMC](https://github.com/eic/eic-smear#creation-of-hepmc-output)
   It is recommended to use EicSmear for Beagle and BNL RootTree files.
- [mcconv](https://eicweb.phy.anl.gov/monte_carlo/mcconv) - new and less reliable, but easier to install and use with convenient python API. mcconv correctly works with GEMC LUND (HallB Clas12) format.


**Example converting CLAS12 file to HepMC**


```bash

    # install mcconv
    pip3 install --upgrade mcconv

    # if there is a sertificate problems
    python3 -m pip install --trusted-host pypi.python.org --trusted-host files.pythonhosted.org --trusted-host pypi.org --user -U mcconv

    # Add home bin directory to the PATH environment variable
    export PATH=~/.local/bin:$PATH

    # convert lund gemc
    mcconv input.txt -i lund_gemc -b 10x100 -v

    # File from example
    mcconv /work/eic/mc/meson_MC/OUTPUTS/0mrad_IR0/nov_2021/pi_n_10.0on100.0_x0.0010-1.0000_q1.0-100.0_lund.dat -v -p 1000
```

<br/>

## 3 Crossing angle and beam effects


In order to apply crossing and beam effects to existing HepMC files [EIC Afterbur](https://github.com/eic/afterburner)
is to be used. The package provides framework independent well validated crossing angle and beam effects C++ library and HepMC file converter (abconv)
for Electron Ion Collider.


**Physics simulated:**

- Crossing angle
- Beam effects (divergence, crabbing kick, etc.)
- Vertex spread (position, time)

The Afterburner uses HepMC2/3 as input and HepMC3 as an output. Beam particles information and the primary vertex must be included in the input hepmc file. 

Run abconv as: 

```
abconv -o out_name_no_ext input_file.hepmc
```


Use `abconv --help` or follow the [instruction of how to use the afterburner](https://github.com/eic/afterburner). 

<br/>

## 4 DD4HEP Simulation

***ddsim vs npsim*** - DD4Hep uses ddsim command to process simulations. EIC has its own wrapper called `npsim` which is essentially `ddsim` but with the EIC physics list, optical photons enabled for certain detector and other properties like this.

[npsim script](https://github.com/eic/npsim/blob/main/src/dd4pod/python/npsim.py)

### Pythia and other EG

```bash
# Detectors live in
# /opt/detectors
# one can select particular configuration as
# source /opt/detector/epic-xxx/bin/thisepic.sh
#
# or one can set the default detector (now points to epic-nightly)
source /opt/detector/epic-main/bin/thisepic.sh

# Run simulation for 1000 events
npsim --compactFile=$DETECTOR_PATH/epic.xml -N=1000 --outputFile=sim_output.edm4hep.root --inputFiles mceg.hepmc
```

### Particle gun

There are at least 2 ways of how to run a particle gun:

-   using ddsim command line
-   using geant macro file and invoke GPS


### Using ddsim

Using ddsim (wrapper around ddsim) command line:

```bash
# set the detector
source /opt/detector/setup.sh

# Electrons with 1MeV - 30GeV fired in all directions, 1000 events
npsim --compactFile=$DETECTOR_PATH/epic.xml -N=1000 --random.seed 1 --enableGun --gun.particle="e-" --gun.momentumMin 1*MeV --gun.momentumMax 30*GeV --gun.distribution uniform --outputFile gun_sim.edm4hep.root

# Pions from defined position and direction
npsim --compactFile=$DETECTOR_PATH/epic.xml -N=1000 --enableGun --gun.particle="pi-" --gun.position "0.0 0.0 1.0*cm" --gun.direction "1.0 0.0 1.0" --gun.energy 100*GeV --outputFile=test_gun.edm4hep.root

# uniform spread inside an angle:
npsim --compactFile=$DETECTOR_PATH/epic.xml -N=2 --random.seed 1 --enableGun --gun.energy 2*GeV --gun.thetaMin 0*deg --gun.thetaMax 90*deg --gun.distribution uniform --outputFile test.edm4hep.root

# run to see all particle gun options
npsim --help
```

### Using Geant4 macros GPS

[GPS stands for General Particle Source (tutorial)](https://geant4-userdoc.web.cern.ch/UsersGuides/ForApplicationDeveloper/html/GettingStarted/generalParticleSource.html)

GPS is configured in Geant4 macro files. An example of such file
[may be found here](https://eicweb.phy.anl.gov/EIC/detectors/athena/-/blob/master/macro/gps.mac)

To run ddsim with GPS you have to add [\--enableG4GPS]{.title-ref} flag
and specify Geant4 macro file:

```bash
npsim --runType run --compactFile=$DETECTOR_PATH/epic.xml --enableG4GPS --macro $DETECTOR_PATH/macro/gps.mac --outputFile gps_example.root
```

## 5 Run reconstruction

[EICrecon documentation](http://eicrecon.epic-eic.org/#/)


```bash
eicrecon -Pjana:debug_plugin_loading=1 -Pjana:nevents=100 -Pjana:timeout=0\
 -Ppodio:output_file=dirc_optical.edm4hep.root\
 -Pdd4hep:xml_files=$DETECTOR_PATH/epic_dirc_only.xml\
 dirc_optical_all.edm4hep.root  
```


### Advanced flags

Here is an advanced example of running eicrecon with common flags with explanation

```bash
eicrecon
-Pplugins=tracking_occupancy,tracking_efficiency
-Pnthreads=1
-Pjana:debug_plugin_loading=1
-Pjana:nevents=100
-Pjana:timeout=0
-Ptracking_efficiency:LogLevel=info
-PTracking:CentralTrackerSourceLinker:LogLevel=info
-PCKFTracking:Trajectories:LogLevel=info
-Ptracking_efficiency:LogLevel=debug
-Ppodio:output_file=/home/romanov/work/data/eicrecon_test/tracking_test_gun.edm4eic.root
-Pdd4hep:xml_files=/home/romanov/eic/soft/detector/main/compiled/epic/share/epic/epic_tracking_only.xml
-Phistsfile=/home/romanov/work/data/eicrecon_test/tracking_test_gun.ana.root
/home/romanov/work/data/eicrecon_test/output.edm4hep.root
```

Flags explained:
```bash
#This flag lists plugins(submodules) needed to run tracking reconstruction and analysis
# These plugins added to the default plugins list. 
# You probably don't need this flag unless you use your plugin
-Pplugins=tracking_efficiency

# Number of parallel threads. Currently only 1 works.
# It is a limitation of an event model IO and will be fixed later
-Pnthreads=1

# Write exactly what happens when plugins are loading. Good during debugging time.
-Pjana:debug_plugin_loading=1

# Removes self "hang" watchdog.
# Needed during debugging if you pause code execution with breakpoints
-Pjana:timeout=0


# Number of events to process.
# (If events needs to be skipped there is also -Pjana:nskip flag)
-Pjana:nevents=100

# xxx:LogLevel - various plugins/factories logging levels
# trace, debug, info, warn, error, critical, off:
# trace    - something very verbose like each hit parameter
# debug    - all information that is relevant for an expert to debug but should not be present in production
# info     - something that will always (almost) get into the global log
# warning  - something that needs attention but results are likely usable
# error    - something bad that makes results probably unusable
# critical - imminent software failure and termination
# Logging explained here:
https://github.com/eic/EICrecon/blob/main/docs/Logging.md


# DD4Hep xml file for the detector describing the geometry.
# Can be set by this flag or env. variables combinations: ${DETECTOR_PATH}/${DETECTOR}.xml
-Pdd4hep:xml_files=...

# Example 1: full path to detector.xml
-Pdd4hep:xml_files=/path/to/dd4hep/epic/epic_tracking_only.xml

# Example2: DETECTOR_PATH env var is set in eic_shell, so it could be
-Pdd4hep:xml_files=${DETECTOR_PATH}/epic_tracking_only.xml


# Alternatively it could be set through environment variables to not to add -Pdd4hep:xml_files every run
# Then -Pdd4hep:xml_files flag is not needed. (!) Note that ".xml" is not needed in ${DETECTOR}
export DETECTOR_PATH="/path/to/dd4hep/epic/"
export DETECTOR="epic_tracking_only"

# This makes tracking output data and input MC particles to be written to the output
-Ppodio:output_include_collections="ReconstructedParticles,TrackParameters,MCParticles"

# There is a centralized file where plugins can save their histograms:
-Phistsfile=/home/romanov/work/data/eicrecon_test/tracking_test_gun.ana.root

# all filenames that doesn't start with -<flag> are interpreted as input files
# So this is an input file path
/home/romanov/work/data/eicrecon_test/output.edm4hep.root
```

### Reconstruction development

Once inside the `eic-shell` you should source the geometry setup script since this is not done by default. Then, clone the `EICrecon` repository and build it:

```bash
source /opt/detector/setup.sh

git clone https://github.com/eic/EICrecon
cmake -S EICrecon -B build
cmake --build build --target install -- -j8
```

Assuming all goes well, `EICrecon` will be installed in the _EICrecon_ directory.
To set you environment up to use it, source the setup script generated by the build:
~~~bash
source EICrecon/bin/eicrecon-this.sh
~~~

Test the installation by running the _eicrecon_ executable with no arguments:

~~~bash
eicrecon

Usage:
    eicrecon [options] source1 source2 ...

Description:
    Command-line interface for running JANA plugins. This can be used to
    read in events and process them. Command-line flags control configuration
    while additional arguments denote input files, which are to be loaded and
    processed by the appropriate EventSource plugin.

Options:
   -h   --help                  Display this message
   -v   --version               Display version information
   -c   --configs               Display configuration parameters
   -l   --loadconfigs <file>    Load configuration parameters from file
   -d   --dumpconfigs <file>    Dump configuration parameters to file
   -b   --benchmark             Run in benchmark mode
   -L   --list_factories        List all the factories without running
   -Pkey=value                  Specify a configuration parameter

Example:
    eicrecon -Pplugins=plugin1,plugin2,plugin3 -Pnthreads=8 inputfile1.txt
~~~

At this point, you can [create your own user plugin](/howto/make_plugin.md) or
[add a new factory (i.e. algorithm)](/howto/add_factory.md).


## How to install Singularity

The [official Singularity installation instructions](https://docs.sylabs.io/guides/4.1/user-guide/quick_start.html) is cumbersome The [official Singularity installation instructions](https://docs.sylabs.io/guides/4.1/user-guide/quick_start.html) is cumbersome 
and an average user might seek simpler alternatives. As of 2024, there is no singularity-container package in the default package
lists of Linux distributions, which means you cannot simply run apt install singularity-container. 
Additionally, be aware of a game also named Singularity that IS actually sits in linux distros PMs - to avoid confusion. 

**The easiest way to install the singularity-container is by directly downloading the package from their GitHub releases**:

https://github.com/sylabs/singularity/releases

For Debian-based systems, you can install it like this:

```bash
sudo apt-get install ./singularity-ce_4.1.3-jammy_amd64.deb
```

It's important to specify "./" so it understands that it refers to a local file.

For RPM-based systems, you can find the corresponding RPM package and use rpm, dnf or yum to install it. For example:

```bash
sudo dnf install ./singularity-ce_4.1.3.el8.x86_64.rpm
```
This command also uses "./" to indicate that the package is locally stored.

Alternatively, for Debian Sid users, the package can be downloaded from:

https://packages.debian.org/sid/amd64/singularity-container/download


