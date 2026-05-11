# Docker Develoment Environment Recommendations
This README outlines two workflows I’ve tested for managing a local development environment for moshpit/pathogenome distributions on an osx machine using Docker. Note that I will change/add things as these processes are improved and fine-tuned with more widespread use.

---------------

## Option 1: Docker Desktop with a base conda container & mounted conda/repo dirs as volumes

This option pulls a pre-built image from conda-forge and mounts all relevant conda subdirs as volumes, which will persist across sessions.

I’ll start by launching Docker desktop so that it’s running in the background. Then, before pulling the conda-forge image, I’ll mount the two necessary conda subdirs as volumes:
```
docker volume create conda-pkgs
docker volume create conda-envs
```
Now I’ll pull the conda-forge image and launch a new container that uses the volumes I just created, as well as my local repositories directory as a work dir, so that I can access those local repos within the container:
```
docker run --rm -it --platform linux/amd64 --user 0:0 --entrypoint '' \
  -v ~/Documents/repositories:/work \
  -v conda-pkgs:/opt/conda/pkgs \
  -v conda-envs:/opt/conda/envs \
  condaforge/linux-anvil-x86_64:alma8 bash
```

Quick descriptions of each of the parameters above:
- `--rm` deletes the container when the session is ended. I find this helps keep things tidy in Docker Desktop, so I don't have a bunch of containers sitting around taking up space.
- `-it` keeps stdin open and provides an interactive terminal, which essentially provides me with a normal shell session.
- `--platform linux/amd64` forces the container to run as linux x86_64 (so there isn't any platform mismatch when creating conda envs that expect intel and not ARM).
- `--user 0:0` run as root inside the container. 0:0 sets UID & GID as zero, respectively. This is what allows me to bypass permission issues when writing to `/opt/conda` and managing envs. Mamba issue [xref](https://github.com/mamba-org/micromamba-docker/issues/445#issuecomment-1993862350) I found that helped me figure this bit out.
- `--entrypoint ''` clears the image's default entrypoint. This was important for this conda-forge image because their default entrypoint sets the user as 'conda'; I ran into permissions issues under this issue which required me to enter a password when attempting to run any `sudo` commands. Snippet [xref](https://github.com/conda-forge/docker-images/blob/6c9f5689754da86080c2c88bc8271493e6f370a9/scripts/entrypoint#L11) from their entrypoint where this is set.
- `-v ~/Documents/repositories:/work` mounts my local repositories directory into the container under `/work`. This could be modified to whatever local repository/directory you wanted access to within the container, and could be named anything.
- `-v conda-pkgs:/opt/conda/pkgs` mounts the Docker volume created above within conda's package cache directory, which is what preserves downloaded packages across container sessions. This will improve new environment solve time for any packages that can be reused across multiple conda env installs.
- `-v conda-envs:/opt/conda/envs` mounts the Docker volume created above within conda's environment directory, which is what preserves any conda environments that are created across container sessions.
- `condaforge/linux-anvil-x86_64:alma8` I picked this image because this is a conda-forge linux build based on AlmaLinux 8 (which has a modern version of glibc=2.27).
- `bash` starts an interactive bash shell after the container is launched - this could be replaced with any subshell of your preference.

Once I have the container running, I will have a split-shell terminal for the following workflow:
- Within my containerized bash shell, I create the conda environment of my choosing and then activate that environment for running unit tests/testing development on the relevant plugin(s).
- Within my local shell, I have VS Code open for modifying the working plugin repositor(ies) and (when I'm ready) pushing up changes to my remotes.

Note that this workflow is effective because I've mounted the location of my local repositories as a volume within the container. So I have dual access to these directories inside the container for testing, and outside of the container for active development and pushing up my changes to Github.

## Option 2: Dockerfile.dev (modified Docker image from the rachis Dockerfile.base image)

This option likely requires more user testing to confirm development workflow efficacy, but should be a viable replacement for the base dockerfile for the purposes of developer use and permissions.

**Dockerfile.dev**
```
FROM condaforge/miniforge:latest

ARG DISTRO
ARG EPOCH=latest
ARG DISTRO_SUBDIR=passed

ENV PATH /opt/conda/envs/rachis-${DISTRO}-${EPOCH}/bin:$PATH
ENV LC_ALL C.UTF-8
ENV LANG C.UTF-8
ENV MPLBACKEND agg
ENV UNIFRAC_USE_GPU N
ENV HOME /home/qiime2
ENV XDG_CONFIG_HOME /home/qiime2

RUN conda update -q -y conda
RUN conda install -q -y wget
RUN apt-get install -y procps

COPY ${EPOCH}/${DISTRO}/${DISTRO_SUBDIR}/rachis-${DISTRO}-linux-64-conda.yml dev-env.yml
RUN conda env create -n rachis-${DISTRO}-${EPOCH} --file dev-env.yml \
 && conda clean -a -y \
 && groupadd -g 1000 qiime2 \
 && useradd -m -u 1000 -g 1000 -s /bin/bash qiime2 \
 && mkdir -p /data /home/qiime2/.cache \
 && chown -R qiime2:qiime2 /opt/conda /data /home/qiime2 \
 && rm dev-env.yml

RUN /bin/bash -c "source activate rachis-${DISTRO}-${EPOCH}"
ENV CONDA_PREFIX /opt/conda/envs/rachis-${DISTRO}-${EPOCH}/
USER qiime2
RUN qiime dev refresh-cache
RUN python -c "from qiime2.sdk.parallel_config import get_vendored_config; get_vendored_config()"
RUN echo "source activate rachis-${DISTRO}-${EPOCH}" >> /home/qiime2/.bashrc
RUN echo "source tab-qiime" >> /home/qiime2/.bashrc

# HACK: regression in libc 2.41 causes:
# libssu_nv_avx2.so: cannot enable executable stack as shared object requires: Invalid argument
ENV GLIBC_TUNABLES glibc.rtld.execstack=2

VOLUME ["/data"]
WORKDIR /data
```
Primary differences between this and `Dockerfile.base`:
- `EPOCH` and `DISTRO_SUBDIR` are set to pull the latest dev env file vs release env file.
- A `qiime2` user/group is set at 1000:1000 with ownership of `/opt/conda`, `/data` and `/home/qiime2`, and cache/init steps are run under this user.
- Shell activation is written to `/home/qiime2/.bashrc`.

These changes allow for conda envs to be created or modified under `/opt/conda` and a mounted repository checkout under `/data` without running as root.
