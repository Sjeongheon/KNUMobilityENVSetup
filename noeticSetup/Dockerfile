###########################################
# Base image
###########################################
FROM osrf/ros:noetic-desktop-full AS base

ARG DEBIAN_FRONTEND=noninteractive
ARG USERNAME=ros
ARG USER_UID=1000
ARG USER_GID=${USER_UID}

# Install basic development dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    sudo \
    git \
    vim \
    bash-completion \
    build-essential \
    cmake \
    can-utils \
    python3-rosdep \
    python3-rosinstall \
    python3-rosinstall-generator \
    python3-wstool \
    ninja-build \
    stow \
    libglvnd0 \
    libgl1 \
    libglx0 \
    libegl1 \
    libxext6 \
    libx11-6 \
  && rm -rf /var/lib/apt/lists/*

# Initialize rosdep
RUN if [ ! -f /etc/ros/rosdep/sources.list.d/20-default.list ]; then \
      rosdep init; \
    fi \
  && rosdep update

# Create a non-root user
RUN groupadd --gid "${USER_GID}" "${USERNAME}" \
  && useradd \
      --uid "${USER_UID}" \
      --gid "${USER_GID}" \
      -m \
      -s /bin/bash \
      "${USERNAME}" \
  && echo "${USERNAME} ALL=(root) NOPASSWD:ALL" \
      > "/etc/sudoers.d/${USERNAME}" \
  && chmod 0440 "/etc/sudoers.d/${USERNAME}"

# Environment variables for NVIDIA Container Runtime
ENV NVIDIA_VISIBLE_DEVICES=all \
    NVIDIA_DRIVER_CAPABILITIES=graphics,utility,compute \
    QT_X11_NO_MITSHM=1


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

# libabsl-dev is not available through rosdep on Ubuntu Focal
RUN sed -i \
    's#<depend>libabsl-dev</depend>#<!-- <depend>libabsl-dev</depend> -->#' \
    src/cartographer/package.xml

# Install Cartographer dependencies
RUN apt-get update \
  && rosdep install \
      --from-paths src \
      --ignore-src \
      --rosdistro="${ROS_DISTRO}" \
      -y \
  && rm -rf /var/lib/apt/lists/*

# Remove conflicting ROS Abseil package if installed
RUN if dpkg-query -W "ros-${ROS_DISTRO}-abseil-cpp" >/dev/null 2>&1; then \
      apt-get update \
      && apt-get remove -y "ros-${ROS_DISTRO}-abseil-cpp" \
      && rm -rf /var/lib/apt/lists/*; \
    fi

# Install Abseil and build Cartographer
RUN src/cartographer/scripts/install_abseil.sh \
  && /bin/bash -lc \
    "source /opt/ros/${ROS_DISTRO}/setup.bash \
    && catkin_make_isolated \
        --install \
        --use-ninja \
        -j$(nproc)"


###########################################
# Cartographer + ugv_sdk + scout_ros
###########################################
FROM cartographer AS scout

# Install SCOUT MINI dependencies
RUN apt-get update \
  && apt-get install -y --no-install-recommends \
    libasio-dev \
    ros-noetic-teleop-twist-keyboard \
  && rm -rf /var/lib/apt/lists/*

# Create catkin workspace
WORKDIR /home/${USERNAME}/catkin_ws/src

# Clone SCOUT MINI ROS packages
RUN git clone --depth 1 \
      https://github.com/agilexrobotics/ugv_sdk.git \
  && git clone --depth 1 \
      https://github.com/agilexrobotics/scout_ros.git

WORKDIR /home/${USERNAME}/catkin_ws

# Install workspace dependencies and build
RUN apt-get update \
  && rosdep install \
      --from-paths src \
      --ignore-src \
      --rosdistro="${ROS_DISTRO}" \
      -y \
  && rm -rf /var/lib/apt/lists/* \
  && /bin/bash -lc \
    "source /opt/ros/${ROS_DISTRO}/setup.bash \
    && source /opt/cartographer_ws/install_isolated/setup.bash \
    && catkin_make -j$(nproc)" \
  && chown -R "${USERNAME}:${USERNAME}" \
      "/home/${USERNAME}/catkin_ws"

# Load ROS workspaces automatically
RUN printf \
    '\nsource /opt/ros/%s/setup.bash\nsource /opt/cartographer_ws/install_isolated/setup.bash --extend\nsource /home/%s/catkin_ws/devel/setup.bash --extend\n' \
    "${ROS_DISTRO}" \
    "${USERNAME}" \
    >> "/home/${USERNAME}/.bashrc" \
  && chown "${USERNAME}:${USERNAME}" \
      "/home/${USERNAME}/.bashrc"

USER ${USERNAME}

WORKDIR /home/${USERNAME}/catkin_ws

CMD ["bash"]