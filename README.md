# CCUR Documentation

## A faire

- Parametrisation ORCA sur le ccur - files d'attentes - memoires
- Pbm droit multi coeurs
- Lancer un des scripts de bader : `[/home/fhoareau/Emilie_Boyer/From_Arnaud_Marvilliers/Hex_A0a_B0a_C1a`

## Tips

### Changement du prompt

Pour mettre le terminal sur le ccur comme sur bader, il suffit d'ajouter la ligne suivant dans le ficher **~/.bashrc** :

```export PS1='\h | \w % '```

Ne pas oublier de sourcer le fichier .bashrc ( "`source ~/.bashrc`" ).

### Connexion

Accès uniquement pas clé publique ssh. Envoyer sa clé publique à ccur@support.univ-reunion.fr.

```bash
ssh -X username@ccur-frontal.univ.run
ssh -X username@10.82.80.222
```

Attention ne pas oublier l'option `-X` auquel cas GaussView ne peut fonctionner.

### Environement modules

- `module avail` : affiche la liste des modules disponibles
- `module load <module>` : charge le module
- `module list` : affche tous les modules actuellement chargés
- `module unload <module>` : décharge le module
- `module purge` : décharge tous les modules de l’environnement
- `module help <module>` : affiche les informations spécifiques d’un module
- `module show <module>` : affiche la liste des modifications effectuées par un module

## Orca

### Exemple d'input

Exemple ORCA avec un fichier d'entrée `inputfile.inp` sur H2O :

```orca
! B3LYP def2-SVP Opt
# My first ORCA calculation
*xyz 0 1
O        0.000000000      0.000000000      0.000000000
H        0.000000000      0.759337000      0.596043000
H        0.000000000     -0.759337000      0.596043000
*
```

### Exemple simple

1- Charger le module Orca :

```bash
module load applications/orca/3.0.2
```

2- Lancer le programme :

```bash
orca inputfile.inp > output.out &
```

### Exemple parallèle

Pour la **version 3** d'ORCA, on utilise OpenMPI version **1.6.5** (chargée par defaut avec ORCA). Pour lancer ORCA en parallèle, il ne faut pas lancer le programme avec la commande `mpirun`.

Utiliser le mot clé `!PalX` dans le fichier d'entrée pour dire à ORCA de commencer une tache multi process. Par exemple, pour commencer un job sur 4 coeurs, le fichier d'entrèe devrait contenir ce code:

```orca
! B3LYP def2-SVP  Opt PAL4
```

ou en utilisant les options en block :

```orca
%pal
nprocs 4
end
```

On peut appeler ORCA mais cette fois avec le chemin entier du programme :

```bash
/gpfs/programs/applications/orca/3.0.2/orca inputfilepar.inp > output.out
```

### Exemple avec gestionnaire de queue

```bash
#!/bin/bash
#PBS -l nodes=1:ppn=8
#PBS -q short

# Usage of this script:
#qsub job-orca.sh -N jobname where jobname is the name of your ORCA inputfile (jobname.inp) without the .inp extension

# Jobname below is set automatically when using "qsub job-orca.sh -N jobname". Can alternatively be set manually here. Should be the name of the inputfile without extension (.inp or whatever).
export job=$PBS_JOBNAME

#Setting OPENMPI paths here:
export PATH=/users/home/user/openmpi/bin:$PATH
export LD_LIBRARY_PATH=/users/home/user/openmpi/lib:$LD_LIBRARY_PATH

# Here giving the path to the ORCA binaries and giving communication protocol
export orcadir=/users/home/user/orca_4_0_1_linux_x86-64_openmpi202
export RSH_COMMAND="/usr/bin/ssh -x"
export PATH=$orcadir:$PATH

# Creating local scratch folder for the user on the computing node. /scratch directory must exist.
if [ ! -d /scratch/$USER ]
then
  mkdir -p /scratch/$USER
fi
tdir=$(mktemp -d /scratch/$USER/orcajob__$PBS_JOBID-XXXX)

# Copy only the necessary stuff in submit directory to scratch directory. Add more here if needed.
cp $PBS_O_WORKDIR/*.inp $tdir/
cp $PBS_O_WORKDIR/*.gbw $tdir/
cp $PBS_O_WORKDIR/*.xyz $tdir/

# Creating nodefile in scratch
cat ${PBS_NODEFILE} > $tdir/$job.nodes

# cd to scratch
cd $tdir

# Copy job and node info to beginning of outputfile
echo "Job execution start: $(date)" >> $PBS_O_WORKDIR/$job.out
echo "Shared library path: $LD_LIBRARY_PATH" >> $PBS_O_WORKDIR/$job.out
echo "PBS Job ID is: ${PBS_JOBID}" >> $PBS_O_WORKDIR/$job.out
echo "PBS Job name is: ${PBS_JOBNAME}" >> $PBS_O_WORKDIR/$job.out
cat $PBS_NODEFILE >> $PBS_O_WORKDIR/$job.out

#Start ORCA job. ORCA is started using full pathname (necessary for parallel execution). Output file is written directly to submit directory on frontnode.
$orcadir/orca $tdir/$job.inp >> $PBS_O_WORKDIR/$job.out

# ORCA has finished here. Now copy important stuff back (xyz files, GBW files etc.). Add more here if needed.
cp $tdir/*.gbw $PBS_O_WORKDIR
cp $tdir/*.xyz $PBS_O_WORKDIR
```

```bash

#!/bin/bash

## University of New Mexico Center for Advanced Research Computing
## Ryan Johnson, May 13, 2015
## PBS submission script to run orca code

#PBS -l walltime=01:00:00
#PBS -l nodes=2:ppn=8
#PBS -N TestIt

# User defined variables
# Add any files here which are needed by ORCA.
# At a minimum, you need the .inp file, which must be first.
files=( "inp.inp" "inp.xyz" )

cwd=$PBS_O_WORKDIR

export RSH_COMMAND="ssh"

# Load Orca and included openmpi modules
module load orca/4.0.1

# Get the full path to orca (helps with mpi problems)
ORCA_EXEC=$(which orca)

# Internal variables for creating scratch dir
PBS_JOBNUM=$(echo PBS_JOBID | cut -d"." -f1 )
input=${files[0]}
NodeList=( $(cat "$PBS_NODEFILE" | sort | uniq -c) )
ExecHost=$(hostname -s)
JobID=$(echo -n "$PBS_JOBID"|cut -d'.' -f1)

# Choosing scratch location based on machine
# Since wheeler does not have hard drives on compute nodes
case "$HOST" in
    *wheeler*)
        # Define and create the scratch directry
        SCRDIR="/wheeler/scratch/$USER/.orcajobs/$JobID"
        mkdir -p "$SCRDIR" ;;
    *)
        # Define and create the scratch directory
        SCRDIR="/tmp/orca/$(whoami)/$PBS_JOBNUM"
        for (( i=0 ; i < ${#NodeList[@]} ; i++ ))
        do
                ((i++))
                if [[ ${NodeList[$i]} == "$ExecHost" ]] ; then
                        mkdir -p "$SCRDIR"
                else
                        ssh -x -n "${NodeList[$i]}" "mkdir -p $SCRDIR"
                fi
        done  ;;
esac

# Create .nodes file and copy to SCRDIR
cp "$PBS_NODEFILE" "$cwd/${input:0:${#input}-4}.nodes"
cp "$cwd/${input:0:${#input}-4}.nodes" "$SCRDIR"

for file in "${files[@]}"
do
        cp "$cwd/$file" "$SCRDIR/"
done

# cd to scratch dir and start orca, redirect output to the PBS_O_WORKDIR

cd "$SCRDIR"

echo Starting job: "$(date)"
echo Job number: "$(echo "$JobID")"

$ORCA_EXEC "$input" > "$cwd/${input:0:${#input}-4}.out"

# orca finished, copy output to user dir
cp "$SCRDIR"/*.* "$cwd"

# remove scratch dir
for (( i=0 ; i < ${#NodeList[@]} ; i++ ))
do
        ((i++))
        if [[ ${NodeList[$i]} == "$ExecHost" ]] ; then
                rm -rf "$SCRDIR"
        else
                ssh -x -n "${NodeList[$i]}" "rm -rf $SCRDIR"
        fi
done

# All done
echo Finish: "$(date)"
```

## G09 sur le ccur

### Initialisation

```bash
module load applications/gaussian/V9
source $G09PROFILE
g09 < inputFile.com > outputFile.log
```

### PBS

```bash
#!/bin/bash
# nb de chunks et coeurs
#PBS -l select=1:ncpus=8
# walltime
#PBS -l walltime=80:00:00
# #PBS –q longq
# #PBS –j oe
# nom du job
#PBS -N demoG09
# notification email : (b)eginning, (e)nd and (a)bortion
#PBS -m bea
# address email
#PBS -M mathieu.delsaut@univ-reunion.fr
# exporter les variables d’environment à la soumission
#PBS -V

# Infos utiles
echo ------------------------------------------------------
echo -n 'Job is running on node '; cat $PBS_NODEFILE
echo ------------------------------------------------------
echo PBS: qsub is running on $PBS_O_HOST
echo PBS: originating queue is $PBS_O_QUEUE
echo PBS: executing queue is $PBS_QUEUE
echo PBS: working directory is $PBS_O_WORKDIR
echo PBS: execution mode is $PBS_ENVIRONMENT
echo PBS: job identifier is $PBS_JOBID
echo PBS: job name is $PBS_JOBNAME
echo PBS: node file is $PBS_NODEFILE
echo PBS: current home directory is $PBS_O_HOME
echo PBS: PATH = $PBS_O_PATH
echo ------------------------------------------------------

# se placer dans le repertoire de travail
cd $PBS_O_WORKDIR
mkdir –p /gpfs/scratch/$USER/$PBS_JOBID
# export PBS_TMPDIR=/gpfs/scratch/$USER/$PBS_JOBID
export GAUSS_SCRDIR=/scratch/$USER/$PBS_JOBID
# cp input $PBS_TMPDIR
# cd $PBS_TMPDIR

# chargement des modules
module purge
module load applications/gaussian/V9
source $G09PROFILE

# lancer l’exécutable
g09 < /gpfs/scratch/mdelsaut/g09/H2_R2.com > output02.log
cp * $PBS_O_WORKDIR
# rm -rf $PBS_TMPDIR

```

## GaussView

Pour lancer GaussView, il suffit d'executer les commandes suivantes :

```bash
load applications/gaussian/V9
source $G09PROFILE
gv -v
```
