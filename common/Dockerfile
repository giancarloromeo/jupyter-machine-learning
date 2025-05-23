FROM nvidia/cuda:11.2.2-cudnn8-runtime-ubuntu18.04 AS ml-base

LABEL maintainer="Andrei Neagu <neagu@speag.swiss>"

#----------- base-notebook
# https://github.com/jupyter/docker-stacks/blob/0710677ce256317ebd403c21656d9fc145f2bf04/base-notebook/Dockerfile

ARG NB_USER="jovyan"
ARG NB_UID="1000"
ARG NB_GID="100"

USER root

# Install all OS dependencies for notebook server that starts but lacks all
# features (e.g., download as all possible file formats)
ENV DEBIAN_FRONTEND noninteractive
RUN apt-key adv --fetch-keys 'https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/3bf863cc.pub' &&\
    apt-get update && \
    apt-get install -yq --no-install-recommends \
    wget \
    bzip2 \
    ca-certificates \
    sudo \
    locales \
    fonts-liberation \
    run-one \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

RUN echo "en_US.UTF-8 UTF-8" > /etc/locale.gen && \
    locale-gen

# Configure environment
ENV CONDA_DIR=/opt/conda \
    SHELL=/bin/bash \
    NB_USER=$NB_USER \
    NB_UID=$NB_UID \
    NB_GID=$NB_GID \
    LC_ALL=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    LANGUAGE=en_US.UTF-8
ENV PATH=$CONDA_DIR/bin:$PATH \
    HOME=/home/$NB_USER

# Copy a script that we will use to correct permissions after running certain commands
COPY src/fix-permissions /usr/local/bin/fix-permissions
RUN chmod a+rx /usr/local/bin/fix-permissions

# Enable prompt color in the skeleton .bashrc before creating the default NB_USER
RUN sed -i 's/^#force_color_prompt=yes/force_color_prompt=yes/' /etc/skel/.bashrc

# Create NB_USER wtih name jovyan user with UID=1000 and in the 'users' group
# and make sure these dirs are writable by the `users` group.
RUN echo "auth requisite pam_deny.so" >> /etc/pam.d/su && \
    sed -i.bak -e 's/^%admin/#%admin/' /etc/sudoers && \
    sed -i.bak -e 's/^%sudo/#%sudo/' /etc/sudoers && \
    useradd -m -s /bin/bash -N -u $NB_UID $NB_USER && \
    mkdir -p $CONDA_DIR && \
    chown $NB_USER:$NB_GID $CONDA_DIR && \
    chmod g+w /etc/passwd && \
    fix-permissions $HOME && \
    fix-permissions $CONDA_DIR

USER $NB_UID
WORKDIR $HOME
ARG PYTHON_VERSION=3.7

# Setup work directory for backward-compatibility
RUN mkdir /home/$NB_USER/work && \
    fix-permissions /home/$NB_USER

ENV MAMBAFORGE_VERSION=23.1.0-1
RUN cd /tmp && \
    wget --quiet "https://github.com/conda-forge/miniforge/releases/download/${MAMBAFORGE_VERSION}/Mambaforge-${MAMBAFORGE_VERSION}-Linux-x86_64.sh" && \
    /bin/bash Mambaforge-${MAMBAFORGE_VERSION}-Linux-x86_64.sh -f -b -p $CONDA_DIR && \
    rm Mambaforge-${MAMBAFORGE_VERSION}-Linux-x86_64.sh && \
    mamba config --system --prepend channels conda-forge && \
    mamba config --system --set auto_update_conda false && \
    mamba config --system --set show_channel_urls true && \
    mamba config --system --set channel_priority strict && \
    mamba config --system --set repodata_threads 5 && \
    mamba install --yes python=$PYTHON_VERSION && \
    mamba list python | grep '^python ' | tr -s ' ' | cut -d '.' -f 1,2 | sed 's/$/.*/' >> $CONDA_DIR/conda-meta/pinned && \
    mamba list tini | grep tini | tr -s ' ' | cut -d ' ' -f 1,2 >> $CONDA_DIR/conda-meta/pinned && \
    mamba install --yes \
    pip \
    'tini=0.18.0' \
    'notebook=6.4.8' \
    'jupyterhub=2.1.1' \
    'jupyterlab=3.2.9' && \
    mamba clean --all -f -y && \
    npm cache clean --force && \
    jupyter notebook --generate-config && \
    rm -rf $CONDA_DIR/share/jupyter/lab/staging && \
    rm -rf /home/$NB_USER/.cache/yarn && \
    fix-permissions /home/$NB_USER

# Copy local files as late as possible to avoid cache busting
COPY src/start.sh src/start-notebook.sh src/start-singleuser.sh /usr/local/bin/
COPY src/jupyter_notebook_config.py /etc/jupyter/

# Fix permissions on /etc/jupyter as root
USER root
RUN fix-permissions /etc/jupyter/

# Switch back to jovyan to avoid accidental container runs as root
USER $NB_UID

#----------- minimal-notebook
# https://github.com/jupyter/docker-stacks/blob/0710677ce256317ebd403c21656d9fc145f2bf04/minimal-notebook/Dockerfile

USER root

# Install all OS dependencies for fully functional notebook server
RUN apt-get update && apt-get install -yq --no-install-recommends \
    build-essential \
    emacs \
    git \
    inkscape \
    jed \
    libsm6 \
    libxext-dev \
    libxrender1 \
    lmodern \
    netcat \
    python-dev \
    # ---- nbconvert dependencies ----
    texlive-xetex \
    texlive-fonts-recommended \
    texlive-generic-recommended \
    # Optional dependency
    texlive-fonts-extra \
    # ----
    tzdata \
    unzip \
    nano \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Switch back to jovyan to avoid accidental container runs as root
USER $NB_UID

#------------- Additional Steps (not taken from jupyter-stacks)
#

USER root

# provides the glxgears for manual testing that a display can be opened etc.
RUN apt-get update && apt-get install -y --no-install-recommends \
    mesa-utils 

# TODO: CR: Try to understand why this is needed
RUN chmod a+rx /usr/local/bin/start-notebook.sh

# Switch back to jovyan to avoid accidental container runs as root
USER $NB_UID



# --------------------------------------------------------------------
# --------------------------------------------------------------------

# TODO: Newest image does not build well jupyterlab extensions
## ARG JUPYTER_MINIMAL_VERSION=54462805efcb@sha256:41c266e7024edd7a9efbae62c4a61527556621366c6eaad170d9c0ff6febc402

ENV JUPYTER_ENABLE_LAB="yes"
ENV NOTEBOOK_TOKEN="simcore"
ENV NOTEBOOK_BASE_DIR="$HOME/work"

USER root

# ffmpeg for matplotlib anim & dvipng for latex labels
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
    ffmpeg \
    dvipng \
    bc \
    zip \
    vim \
    gosu \
    && rm -rf /var/lib/apt/lists/*

RUN pip --no-cache --quiet install --upgrade \
    pip \
    setuptools \
    wheel

USER $NB_UID

# jupyter customizations
# https://github.com/jupyterlab/jupyterlab-git/releases
#
RUN mamba install --quiet --yes \
    jupyterlab-git \
    # https://github.com/jupyterlab/jupyterlab-latex/
    jupyterlab_latex \
    && mamba install -c conda-forge \
    ipyevents==2.0.1 \
    ipympl==0.8.8 \
    && mamba clean --all -f -y \
    && jupyter lab build -y \
    && jupyter lab clean -y \
    && npm cache clean --force \
    && rm -rf /home/$NB_USER/.cache/yarn \
    && rm -rf /home/$NB_USER/.node-gyp \
    && fix-permissions $CONDA_DIR \
    && fix-permissions /home/$NB_USER

# sidecar functionality -------------------------------------

ENV PYTHONPATH="/src:$PYTHONPATH"


EXPOSE 8888

CMD ["start-notebook.sh"]
ENTRYPOINT [ "/bin/bash", "/docker/run.bash" ]


# Install voila parts. This may done higher up in the chain such that the other flavors also have it
RUN pip install voila && \
    jupyter serverextension enable voila && \
    jupyter server extension enable voila

USER root
# installing python via pyenv
ARG PYTHON_VERSION=3.9.12
ENV PYENV_ROOT $HOME/.pyenv
ENV PATH $PYENV_ROOT/shims:$PYENV_ROOT/bin:$PATH
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    make \
    build-essential \
    libssl-dev \
    zlib1g-dev \
    libbz2-dev \
    libreadline-dev \
    libsqlite3-dev \
    wget \
    curl \
    llvm \
    libncurses5-dev \
    xz-utils \
    tk-dev \
    libxml2-dev \
    libxmlsec1-dev \
    libffi-dev \
    liblzma-dev \
    && apt-get clean && rm -rf /var/lib/apt/lists/*
USER $NB_USER
RUN git clone https://github.com/pyenv/pyenv.git ${HOME}/.pyenv && \
    git clone https://github.com/pyenv/pyenv-virtualenv.git $(pyenv root)/plugins/pyenv-virtualenv && \
    pyenv install ${PYTHON_VERSION} && \
    pyenv virtualenv ${PYTHON_VERSION} lab-venv && \
    ${HOME}/.pyenv/versions/lab-venv/bin/python --version && \
    ${HOME}/.pyenv/versions/lab-venv/bin/pip install --upgrade pip ipykernel && \
    ${HOME}/.pyenv/versions/lab-venv/bin/python -m ipykernel install --user --name kpy39 --display-name "Python (${PYTHON_VERSION})" && \
    echo y | jupyter kernelspec uninstall python3

USER root
# Copying boot scripts
COPY --chown=$NB_UID:$NB_GID docker /docker

# get replaced when notebook starts
ENV DY_SIDECAR_PATH_INPUTS="$HOME/.hidden_inputs" \
    DY_SIDECAR_PATH_OUTPUTS="$HOME/.hidden_outputs" \
    DY_BOOT_OPTION_BOOT_MODE=0

# copy README and CHANGELOG
COPY --chown=$NB_UID:$NB_GID README.ipynb ${NOTEBOOK_BASE_DIR}/README.ipynb

ENV LD_LIBRARY_PATH="${LD_LIBRARY_PATH}:/usr/local/cuda-11.2/lib64"

RUN echo 'source /home/${NB_USER}/.pyenv/versions/lab-venv/bin/activate' >> "/home/${NB_USER}/.bashrc"