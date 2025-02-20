ARG V_BASE=3.3.0

FROM opendatacube/geobase-builder:${V_BASE} as env_builder
ARG py_env_path=/env

ENV PATH=${py_env_path}/bin:$PATH

# install any libs - needed for stage1 dependency
COPY requirements0.txt constraints0.txt /conf/
RUN echo "Building python prereq: stage 0" \
  && cat /conf/requirements0.txt \
  && env-build-tool new /conf/requirements0.txt /conf/constraints0.txt ${py_env_path}

COPY requirements.txt constraints.txt nobinary.txt /conf/
RUN echo "Building python environment: stage 1" \
  && cat /conf/requirements.txt \
  && NOBINARY=/conf/nobinary.txt env-build-tool extend /conf/requirements.txt /conf/constraints.txt ${py_env_path}

COPY requirements-jupyter.txt constraints-jupyter.txt /conf/
RUN echo "Building python environment: stage Jupyter" \
  && cat /conf/requirements-jupyter.txt \
  && env-build-tool extend /conf/requirements-jupyter.txt /conf/constraints-jupyter.txt ${py_env_path}

# NOTE - JupyterLab extensions can now be distributed as prebuilt extensions and available to PyPI
RUN echo "Enable jupyter lab extensions" \
  && jupyter labextension install jupyterlab-theme-toggle \
  && jupyter labextension list \
  && echo "...done"

RUN echo "Enable server extensions" \
  && jupyter server extension enable --py jupyterlab_iframe --sys-prefix \
  && jupyter server extension enable --py jupyter_resource_usage  --sys-prefix \
  && jupyter server extension list \
  && echo "...done"

COPY requirements-odc-static.txt constraints-odc-static.txt /conf/
RUN echo "Adding odc-dependencies (static)" \
  && env-build-tool extend /conf/requirements-odc-static.txt /conf/constraints-odc-static.txt ${py_env_path}

FROM opendatacube/geobase-runner:${V_BASE}

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update -y \
  && apt-get install -y --fix-missing --no-install-recommends \
     # developer convenience
     postgresql-client-12 \
     postgresql-12 \
     less \
     wget \
     curl \
     vim \
     tmux \
     htop \
     fish \
     tig \
     git \
     jq \
     xz-utils \
     zip \
     unzip \
     file \
     time \
     openssh-client \
     graphviz \
     sudo \
     iproute2 \
     iputils-ping \
     net-tools \
     simpleproxy \
     rsync \
     libtiff-tools \
    # rgsislib dependencies
     libgsl-dev \
     libcgal-dev \
     libboost-dev \
     libmuparser2v5 \
     libboost-system1.71.0 \
     libboost-filesystem1.71.0 \
     libboost-date-time1.71.0 \
    # for cython to work need compilers
     build-essential \
    # for pyRAT install or something
     libfftw3-dev \
     liblapack-dev \
    # install ffmpeg the normal way
     ffmpeg \
  && rm -rf /var/lib/apt/lists/*

# Install Tini
COPY --from=env_builder /bin/tini /bin/tini

ARG nb_user=jovyan
ARG nb_uid=1000
ARG nb_gid=100

RUN useradd -m -s /bin/bash -N -g $nb_gid -u $nb_uid $nb_user

# Copy python env
ARG py_env_path=/env
COPY --chown=1000:100 --from=env_builder $py_env_path $py_env_path

ENV LC_ALL=C.UTF-8
ENV SHELL=bash
# Put `/usr/local/bin` before env to allow overrides in there
ENV PATH=/usr/local/bin:${py_env_path}/bin:$PATH

# Use `--build-arg PURGE_CACHE_ODC=$(date)` to force re-installation of ODC dependencies
# without the need to "edit" requirements-odc.txt
ARG PURGE_CACHE_ODC=N
COPY requirements-odc.txt constraints-odc.txt /conf/
USER $nb_user
RUN echo "Adding odc-dependencies" \
  && if [ "${PURGE_CACHE_ODC}" != "N" ]; then echo "Forced docker to purge cache for this step!"; fi \
  && ${py_env_path}/bin/pip install \
    --no-cache-dir \
    --requirement=/conf/requirements-odc.txt \
    -c /conf/constraints-odc.txt


USER root

COPY assets/sync_repo assets/with_bootstrap assets/jupyterhub-singleuser /usr/local/bin/
COPY assets/overrides.json $py_env_path/share/jupyter/lab/settings/

WORKDIR "/home/$nb_user"

ARG WITH_SUDO="no"
RUN if [ "$WITH_SUDO" = "yes" ]; then \
  echo "${nb_user} ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers \
;fi

ARG BUILD_INFO=""
RUN if [ -z "${BUILD_INFO}" ]; then \
  BUILD_INFO="$(date)" \
;fi \
  && echo "${BUILD_INFO}" > /build.info

USER $nb_user

ENTRYPOINT ["/bin/tini", "-s", "--", "with_bootstrap"]
CMD ["jupyter", "lab", \
"--ip=0.0.0.0", \
"--port=9988", \
"--no-browser"]
