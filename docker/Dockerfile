# ROS distribution to use
ARG ROS_DISTRO=noetic

###############################################################################################
# Base Image for #
###############################################################################################

FROM osrf/ros:${ROS_DISTRO}-desktop-full AS base
ENV ROS_DISTRO=${ROS_DISTRO}
SHELL ["/bin/bash", "-c"]

# Install basic apt packages
RUN apt update && \
  DEBIAN_FRONTEND=noninteractive \
  apt install --no-install-recommends -y \
  build-essential \
  cmake \
  cppcheck \
  curl \
  git \
  gnupg \
  libeigen3-dev \
  libgles2-mesa-dev \
  lsb-release \
  pkg-config \
  protobuf-compiler \
  python3-dbg \
  python3-pip \
  python3-scipy \
  python3-venv \
  qtbase5-dev \
  ruby \
  software-properties-common \
  sudo \
  wget \
  && apt clean

# Install CUDA Toolkit
RUN wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
RUN dpkg -i cuda-keyring_1.1-1_all.deb
RUN apt-get update && apt-get install -y --no-install-recommends \
    cuda-toolkit-12.2 \
    nvidia-gds \
    && rm -rf /var/lib/apt/lists/*
ENV PATH="/usr/local/cuda-12.2/bin:${PATH}"
ENV LD_LIBRARY_PATH="/usr/local/cuda-12.2/lib64:${LD_LIBRARY_PATH}"

# Install python requirements
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt --no-cache-dir

# Install some 'standard' ROS packages and utilities.
RUN apt update \
  && apt install -y --no-install-recommends \
  python3-catkin-tools \
  python3-rosdep \
  python3-rosinstall \
  python3-rosinstall-generator \
  python3-vcstool \
  ros-${ROS_DISTRO}-rqt \
  ros-${ROS_DISTRO}-rqt-common-plugins \
  ros-${ROS_DISTRO}-rqt-robot-plugins \
  ros-${ROS_DISTRO}-rviz \
  && apt clean

# Create Catkin workspace 
RUN mkdir -p /underlay_ws/src 
WORKDIR /underlay_ws/src

# Copy the dependencies file
COPY dependencies.repos ./
RUN vcs import < dependencies.repos

RUN source /opt/ros/noetic/setup.bash \
 && catkin init 

# Build the base Catkin workspace
RUN cd /underlay_ws \
 && source /opt/ros/noetic/setup.bash \
 #&& rosdep install -y --from-paths src --ignore-src \
 && catkin build

# Remove display warnings
RUN mkdir /tmp/runtime-root
ENV XDG_RUNTIME_DIR "/tmp/runtime-root"
ENV NO_AT_BRIDGE 1

# Set up the work directory and entrypoint
WORKDIR /underlay_ws
COPY ./docker/entrypoint.sh /

# Alias python3 as python
RUN ln -s /usr/bin/python3 /usr/bin/python

ENTRYPOINT [ "/entrypoint.sh" ]

###############################################################################################
# Overlay Image #
###############################################################################################
FROM base AS overlay

# Create an overlay Catkin workspace
RUN source /opt/ros/noetic/setup.bash \
 && mkdir -p /overlay_ws/src \
 && cd /overlay_ws \ 
 && catkin init
#COPY ./some_folder/ /overlay_ws/src/some_folder/
#COPY ./some_other_folder/ /overlay_ws/src/some_other_folder/
RUN source /opt/ros/noetic/setup.bash \
 && catkin build

# Set up the work directory and entrypoint
WORKDIR /overlay_ws
COPY ./docker/entrypoint.sh /
ENTRYPOINT [ "/entrypoint.sh" ]

################################################################################################
## Development Image #
################################################################################################
FROM overlay AS dev

# Dev container arguments
ARG USERNAME=devuser
ARG UID=1000
ARG GID=${UID}

# Install extra tools for development
RUN apt-get update && apt-get install -y --no-install-recommends \
 gdb gdbserver nano

# Create new user and home directory
RUN groupadd --gid $GID $USERNAME \
 && useradd --uid ${GID} --gid ${UID} --create-home ${USERNAME} \
 && echo ${USERNAME} ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/${USERNAME} \
 && chmod 0440 /etc/sudoers.d/${USERNAME} \
 && mkdir -p /home/${USERNAME} \
 && chown -R ${UID}:${GID} /home/${USERNAME}

# Set the ownership of the overlay workspace to the new user
RUN chown -R ${UID}:${GID} /overlay_ws/


# Set the user and source entrypoint in the user's .bashrc file
USER ${USERNAME}
RUN echo "source /entrypoint.sh" >> /home/${USERNAME}/.bashrc
