###########################################
# base image 
###########################################
FROM osrf/ros:melodic-desktop-full AS base

RUN apt-get update && apt-get install -y \
    python-rosdep \
    python-rosinstall \
    python-rosinstall-generator \
    python-wstool \
    python-pip \
    python-pep8 \
    python-autopep8 \
    pylint \
    build-essential \
    bash-completion \
    git \
    vim \
    cmake \
    can-utils \
  && rm -rf /var/lib/apt/lists/* \
  && rosdep init || echo "rosdep already initialized"

# Setup environment
ENV LD_LIBRARY_PATH=/opt/ros/melodic/lib
ENV ROS_DISTRO=melodic
ENV ROS_ROOT=/opt/ros/melodic/share/ros
ENV ROS_PACKAGE_PATH=/opt/ros/melodic/share
ENV ROS_MASTER_URI=http://localhost:11311
ENV ROS_PYTHON_VERSION=
ENV ROS_VERSION=1
ENV PATH=/opt/ros/melodic/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ENV ROSLISP_PACKAGE_DIRECTORIES=
ENV PYTHONPATH=/opt/ros/melodic/lib/python2.7/dist-packages
ENV PKG_CONFIG_PATH=/opt/ros/melodic/lib/pkgconfig
ENV ROS_ETC_DIR=/opt/ros/melodic/etc/ros
ENV CMAKE_PREFIX_PATH=/opt/ros/melodic

# Create a non-root user
ARG USERNAME=ros
ARG USER_UID=1000
ARG USER_GID=$USER_UID

RUN groupadd --gid $USER_GID $USERNAME \
  && useradd -s /bin/bash --uid $USER_UID --gid $USER_GID -m $USERNAME \
  && mkdir /home/$USERNAME/.config && chown $USER_UID:$USER_GID /home/$USERNAME/.config

# Set up sudo
RUN apt-get update \
  && apt-get install -y sudo \
  && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME\
  && chmod 0440 /etc/sudoers.d/$USERNAME \
  && rm -rf /var/lib/apt/lists/*

# Set autocompletion
RUN apt-get update && apt-get install -y git-core bash-completion \
  && echo "if [ -f /opt/ros/${ROS_DISTRO}/setup.bash ]; then source /opt/ros/${ROS_DISTRO}/setup.bash; fi" >> /home/$USERNAME/.bashrc \
  && rm -rf /var/lib/apt/lists/* 

# Install Gazebo
RUN apt-get update && apt-get install -y \
  ros-melodic-gazebo* \
  && rm -rf /var/lib/apt/lists/*

# Expose the nvidia driver to allow opengl 
# Dependencies for glvnd and X11.
RUN apt-get update \
 && apt-get install -y -qq --no-install-recommends \
  libglvnd0 \
  libgl1 \
  libglx0 \
  libegl1 \
  libxext6 \
  libx11-6

# Env vars for the nvidia-container-runtime.
ENV NVIDIA_VISIBLE_DEVICES all
ENV NVIDIA_DRIVER_CAPABILITIES graphics,utility,compute
ENV QT_X11_NO_MITSHM 1


###########################################
# base + cartographer
###########################################
FROM base AS cartographer

# Install cartographer
RUN apt-get update \
  && apt-get install -y -qq \ 
  ninja-build \
  stow \
  libceres-dev \
  libopencv-dev \
  libeigen3-dev \
  libsuitesparse-dev \
  lua5.3 \
  liblua5.3-dev \
  && rm -rf /var/lib/apt/lists/*

# Make cartographer workspace(isolated)
RUN mkdir /home/${USERNAME}/cartographer_ws

WORKDIR /home/${USERNAME}/cartographer_ws

# Initialize cartographer
RUN wstool init src \
 && wstool merge -t src https://raw.githubusercontent.com/cartographer-project/cartographer_ros/master/cartographer_ros.rosinstall \
 && wstool update -t src

# Install cartographer dependency
RUN src/cartographer/scripts/install_abseil.sh

# Build cartographer
RUN catkin_make_isolated --install --use-ninja -j6

# Load & Extend installed environment 
RUN echo "source /root/cartographer_ws/install_isolated/setup.bash --extend" >> ~/.bashrc

###########################################
# cartographer + ugv_sdk + scout_ros
###########################################

FROM cartographer AS scout

RUN mkdir -p /home/${USERNAME}/catkin_ws/src
WORKDIR /home/${USERNAME}/catkin_ws/src

RUN apt-get update \
  && apt-get install -y libasio-dev \
  && rm -rf /var/lib/apt/lists/*

RUN git clone https://github.com/agilexrobotics/ugv_sdk.git \
  && git clone https://github.com/agilexrobotics/scout_ros.git \
  && cd .. \
  && catkin_make -j6