# Second hands-on - Shifter

## Logging in to NERSC

Use ssh to connect to Cori.  The username and password will be on the training account sheet.

```bash
ssh <account>@cori.nersc.gov
```

## Pulling an image

Pull an image using shifterimg.  You can pull a standard image such as Ubuntu or an image you pushed to dockerhub in the previous session.

```bash
shifterimg pull ubuntu:14.04
# OR
shifterimg pull scanon/shanetest:latest
```

## Running an image interactively

Use salloc and shifter to test the image.

```bash
salloc -N 1 -C haswell -p regular --reservation=sc17_shifter --image ubuntu:14.04 -A ntrain
shifter bash
```

You should be able to browse inside the image and confirm that it matches what you pushed to dockerhub earlier.

```bash
ls -l /app
lsb_release -a
```

Once you are done exploring, exit out.
```bash
exit
exit
```

## Submitting a Shifter batch job

Now create a batch submission script and try running a batch job with shifter.  Use vi or your other favorite editor to create the submission script or cat the contents into a file.

```bash
cat << EOF > submit.sl
#!/bin/bash
#SBATCH -N 1 -C haswell
#SBATCH --reservation=sc17_shifter
#SBATCH -p regular
#SBATCH -A ntrain
#SBATCH --image ubuntu:latest

srun -N 1 shifter /app/app.py
EOF
```
Use the Slurm sbatch command to submit the script.

```bash
sbatch ./submit.sl
```

## Running a parallel Python MPI job

It is possible to run MPI jobs in Shifter and obtain native performance.  There are several ways to achieve this. We will demonstrate one approach here.

On your laptop create and push a docker image with MPICH and a sample application installed.


First, create and save a Hello World MPI application.
```code
// Hello World MPI app
#include <mpi.h>

int main(int argc, char** argv) {
    int size, rank;
    char buffer[1024];

    MPI_Init(&argc, &argv);

    MPI_Comm_size(MPI_COMM_WORLD, &size);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

    gethostname(buffer, 1024);

    printf("hello from %d of %d on %s\n", rank, size, buffer);

    MPI_Barrier(MPI_COMM_WORLD);

    MPI_Finalize();
    return 0;
}
```

Next create, a Dockerfile that install MPICH and the application.

```bash
# MPI Dockerfile
FROM nersc/ubuntu-mpi:14.04

ADD helloworld.c /app/

RUN cd /app && mpicc helloworld.c -o /app/hello

ENV PATH=/usr/bin:/bin:/app:/usr/local/bin
```

Build and push the image.
```bash
docker build -t <mydockerid>/hellompi:latest .
docker push <mydockerid>/hellompi:latest
```

Next, return to your Cori login, pull your image down and run it.

```bash
shifterimg pull <mydockerid>/hellompi:latest
#Wait for it to complete
salloc -N 2 -C haswell -p regular -A ntrain --reservation=sc17_shifter --image <mydockerid>/hellompi:latest
# Wait for prepare_compilation_report
# Cori has 32 physical cores per node with 2 hyper-threads per core.  
# So you can run up to 64 tasks per node.
srun -N 2 -n 128 shifter /app/hello
exit
```

If you have your own MPI applications, you can attempt to Docker-ize them using the steps above and run it on Cori.  As a courtesy, limit your job sizes to leave sufficient resources for other participants.  _Don't forget to exit from any "salloc" shells once you are done testing._

## Using Volume mounts

Like Docker, Shifter allows you to mount directories into your container.
The syntax is similar to Docker but uses "--volume".  Here we will mount a
scratch directory into the volume as /data.

```bash
mkdir $SCRATCH/input
echo 3.141592 > $SCRATCH/input/data.txt
shifter --volume $SCRATCH/input:/data --image=ubuntu bash
cat /data/data.txt
```
