###########################################
# Base image
###########################################
FROM osrf/ros:melodic-desktop-full AS base

ARG DEBIAN_FRONTEND=noninteractive

ARG USERNAME=ros
ARG USER_UID=1000
ARG USER_GID=${USER_UID}

# Install basic development dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
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
    sudo \
  && rm -rf /var/lib/apt/lists/*

# Initialize rosdep
RUN if [ ! -f /etc/ros/rosdep/sources.list.d/20-default.list ]; then \
      rosdep init; \
    fi \
  && rosdep update

# Create a non-root user
RUN groupadd --gid ${USER_GID} ${USERNAME} \
  && useradd \
      --uid ${USER_UID} \
      --gid ${USER_GID} \
      -m \
      -s /bin/bash \
      ${USERNAME} \
  && mkdir -p /home/${USERNAME}/.config \
  && chown -R ${USERNAME}:${USERNAME} /home/${USERNAME}

# Set up sudo
RUN echo "${USERNAME} ALL=(root) NOPASSWD:ALL" \
      > /etc/sudoers.d/${USERNAME} \
  && chmod 0440 /etc/sudoers.d/${USERNAME}

# Set ROS environment & autocompletion
RUN echo \
    "if [ -f /opt/ros/${ROS_DISTRO}/setup.bash ]; then \
    source /opt/ros/${ROS_DISTRO}/setup.bash; \
    fi" \
    >> /home/${USERNAME}/.bashrc

# Install Gazebo
RUN apt-get update && apt-get install -y --no-install-recommends \
    ros-melodic-gazebo* \
  && rm -rf /var/lib/apt/lists/*

# Dependencies for NVIDIA/OpenGL and X11
RUN apt-get update \
  && apt-get install -y --no-install-recommends \
    libglvnd0 \
    libgl1 \
    libglx0 \
    libegl1 \
    libxext6 \
    libx11-6 \
  && rm -rf /var/lib/apt/lists/*

# Environment variables for NVIDIA Container Runtime
ENV NVIDIA_VISIBLE_DEVICES=all
ENV NVIDIA_DRIVER_CAPABILITIES=graphics,utility,compute
ENV QT_X11_NO_MITSHM=1


###########################################
# Base + Cartographer
###########################################
FROM base AS cartographer

WORKDIR /opt/cartographer_ws

# Initialize Cartographer workspace
RUN wstool init src \
  && wstool merge -t src \
    https://raw.githubusercontent.com/cartographer-project/cartographer_ros/master/cartographer_ros.rosinstall \
  && wstool update -t src

# Install Cartographer dependencies
RUN apt-get update \
  && rosdep install \
      --from-paths src \
      --ignore-src \
      --rosdistro=${ROS_DISTRO} \
      -y \
  && if dpkg -s "ros-${ROS_DISTRO}-abseil-cpp" > /dev/null 2>&1; then \
       apt-get remove -y "ros-${ROS_DISTRO}-abseil-cpp"; \
     fi \
  && rm -rf /var/lib/apt/lists/*

# Install Abseil
RUN src/cartographer/scripts/install_abseil.sh

# Build Cartographer
RUN /bin/bash -lc \
    "source /opt/ros/${ROS_DISTRO}/setup.bash \
    && catkin_make_isolated \
        --install \
        --use-ninja \
        -j$(nproc)"

# Load Cartographer environment
RUN echo \
    "source /opt/cartographer_ws/install_isolated/setup.bash --extend" \
    >> /home/${USERNAME}/.bashrc


###########################################
# Cartographer + ugv_sdk + scout_ros
###########################################
FROM cartographer AS scout

# Create catkin workspace
RUN mkdir -p /home/${USERNAME}/catkin_ws/src

WORKDIR /home/${USERNAME}/catkin_ws/src

# Install SCOUT MINI dependencies
RUN apt-get update \
  && apt-get install -y --no-install-recommends \
    libasio-dev \
  && rm -rf /var/lib/apt/lists/*

# Clone SCOUT MINI ROS packages
RUN git clone --depth 1 \
      https://github.com/agilexrobotics/ugv_sdk.git \
  && git clone --depth 1 \
      https://github.com/agilexrobotics/scout_ros.git

WORKDIR /home/${USERNAME}/catkin_ws

# Install workspace dependencies
RUN apt-get update \
  && rosdep install \
      --from-paths src \
      --ignore-src \
      --rosdistro=${ROS_DISTRO} \
      -y \
  && rm -rf /var/lib/apt/lists/*

# Build catkin workspace
RUN /bin/bash -lc \
    "source /opt/ros/${ROS_DISTRO}/setup.bash \
    && source /opt/cartographer_ws/install_isolated/setup.bash \
    && catkin_make -j$(nproc)"

# Load workspace automatically
RUN echo \
    "source /home/${USERNAME}/catkin_ws/devel/setup.bash --extend" \
    >> /home/${USERNAME}/.bashrc \
  && chown -R ${USERNAME}:${USERNAME} /home/${USERNAME}

USER ${USERNAME}

WORKDIR /home/${USERNAME}/catkin_ws

CMD ["bash"]