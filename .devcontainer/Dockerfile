FROM ubuntu:jammy

ENV DEBIAN_FRONTEND=noninteractive
ENV LANG=C.UTF-8
ENV LC_ALL=C.UTF-8
ENV ROS_DISTRO=humble

# ── 1. Base tools ─────────────────────────────────────────────────────────────
RUN apt-get update && apt-get install --no-install-recommends -y \
    apt-utils \
    build-essential \
    ca-certificates \
    cmake \
    curl \
    git \
    gnupg2 \
    locales \
    lsb-release \
    nano \
    ninja-build \
    python3-dev \
    python3-pip \
    tar \
    unzip \
    wget \
    libglvnd0 \
    libglx0 \
    libegl1 \
    libxext6 \
    libx11-6 \
    libboost-all-dev \
    && rm -rf /var/lib/apt/lists/*

# ── 2. ROS 2 Humble apt repo ──────────────────────────────────────────────────
RUN curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
      -o /usr/share/keyrings/ros-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
      http://packages.ros.org/ros2/ubuntu $(lsb_release -cs) main" \
      | tee /etc/apt/sources.list.d/ros2.list > /dev/null

# ── 3. Gazebo Garden apt repo (matches PX4 gz_* SITL targets on Ubuntu 22.04) ─
RUN curl -sSL https://packages.osrfoundation.org/gazebo.gpg \
      -o /usr/share/keyrings/pkgs-osrf-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/pkgs-osrf-archive-keyring.gpg] \
      http://packages.osrfoundation.org/gazebo/ubuntu-stable $(lsb_release -cs) main" \
      | tee /etc/apt/sources.list.d/gazebo-stable.list > /dev/null

# ── 4. ROS 2 + Gazebo packages ────────────────────────────────────────────────
RUN apt-get update && apt-get install --no-install-recommends -y \
    ros-${ROS_DISTRO}-ros-base \
    ros-${ROS_DISTRO}-rviz2 \
    ros-${ROS_DISTRO}-rqt* \
    ros-${ROS_DISTRO}-tf2-ros \
    ros-${ROS_DISTRO}-tf2-tools \
    ros-${ROS_DISTRO}-geographic-msgs \
    ros-${ROS_DISTRO}-diagnostic-updater \
    ros-${ROS_DISTRO}-angles \
    ros-${ROS_DISTRO}-ompl \
    ros-${ROS_DISTRO}-behaviortree-cpp-v3 \
    ros-${ROS_DISTRO}-cv-bridge \
    ros-${ROS_DISTRO}-image-transport \
    ros-${ROS_DISTRO}-ros-gz \
    gz-garden \
    python3-colcon-common-extensions \
    python3-rosdep \
    && rm -rf /var/lib/apt/lists/*

# ── 5. rosdep init ────────────────────────────────────────────────────────────
RUN rosdep init && rosdep update

# ── 6. GeographicLib datasets (required by MAVROS) ────────────────────────────
RUN wget -q https://raw.githubusercontent.com/mavlink/mavros/master/mavros/scripts/install_geographiclib_datasets.sh \
    && bash install_geographiclib_datasets.sh \
    && rm install_geographiclib_datasets.sh

# ── 7. PX4 build dependencies ─────────────────────────────────────────────────
RUN apt-get update && apt-get install --no-install-recommends -y \
    astyle \
    libgstreamer1.0-dev \
    libgstreamer-plugins-base1.0-dev \
    gstreamer1.0-plugins-good \
    gstreamer1.0-plugins-bad \
    gstreamer1.0-plugins-ugly \
    libzmq3-dev \
    libxml2-dev \
    sudo \
    && rm -rf /var/lib/apt/lists/*

# empy 3.3.4 required — PX4 build breaks with empy 4.x
RUN pip3 install --no-cache-dir \
    empy==3.3.4 \
    kconfiglib \
    jsonschema \
    jinja2 \
    packaging \
    toml \
    pyros-genmsg

# ── 8. Non-root user ──────────────────────────────────────────────────────────
ARG USERNAME=dev
ARG USER_UID=1000
ARG USER_GID=$USER_UID

RUN groupadd --gid $USER_GID $USERNAME \
    && useradd -s /bin/bash --uid $USER_UID --gid $USER_GID -m $USERNAME \
    && echo "$USERNAME ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME

# ── 9. Clone all source repos ─────────────────────────────────────────────────
WORKDIR /workspace

RUN git clone --recursive --depth 1 https://github.com/kangmin7/PX4-Autopilot.git

RUN mkdir -p ros_ws/src \
    && git clone --depth 1 https://github.com/kangmin7/mavros.git           ros_ws/src/mavros \
    && git clone --depth 1 https://github.com/kangmin7/slam_toolbox-px4.git ros_ws/src/slam_toolbox \
    && git clone --depth 1 https://github.com/kangmin7/navigation2-px4.git  ros_ws/src/navigation2 \
    && git clone --depth 1 https://github.com/kangmin7/m-explore-ros2.git   ros_ws/src/m-explore-ros2

# ── 10. Install all rosdep dependencies up front ──────────────────────────────
WORKDIR /workspace/ros_ws
RUN apt-get update \
    && . /opt/ros/${ROS_DISTRO}/setup.sh \
    && rosdep install --from-paths src --ignore-src -r -y \
         --skip-keys ros-humble-gazebo-ros-pkgs \
    || true \
    && rm -rf /var/lib/apt/lists/*

# ── 10a. Build mavros (sequential, single-threaded) ───────────────────────────
RUN . /opt/ros/${ROS_DISTRO}/setup.sh \
    && colcon build --executor sequential --parallel-workers 1 \
         --packages-up-to mavros \
         --cmake-args -DCMAKE_BUILD_TYPE=Release -DCMAKE_BUILD_PARALLEL_LEVEL=1

# ── 10b. Build slam_toolbox ───────────────────────────────────────────────────
RUN . /opt/ros/${ROS_DISTRO}/setup.sh \
    && . /workspace/ros_ws/install/setup.sh \
    && colcon build --executor sequential --parallel-workers 1 \
         --packages-up-to slam_toolbox \
         --cmake-args -DCMAKE_BUILD_TYPE=Release -DCMAKE_BUILD_PARALLEL_LEVEL=1

# ── 10c. Build navigation2 (largest step — many packages, build one at a time) ─
RUN . /opt/ros/${ROS_DISTRO}/setup.sh \
    && . /workspace/ros_ws/install/setup.sh \
    && colcon build --executor sequential --parallel-workers 1 \
         --packages-up-to navigation2 \
         --cmake-args -DCMAKE_BUILD_TYPE=Release -DCMAKE_BUILD_PARALLEL_LEVEL=1

# ── 10d. Build m-explore-ros2 ─────────────────────────────────────────────────
RUN . /opt/ros/${ROS_DISTRO}/setup.sh \
    && . /workspace/ros_ws/install/setup.sh \
    && colcon build --executor sequential --parallel-workers 1 \
         --packages-select explore_lite multirobot_map_merge \
         --cmake-args -DCMAKE_BUILD_TYPE=Release -DCMAKE_BUILD_PARALLEL_LEVEL=1

# ── 11. Build PX4 SITL (compile only, do not launch simulation) ───────────────
WORKDIR /workspace/PX4-Autopilot
RUN make px4_sitl_default

# ── 12. Shell environment ─────────────────────────────────────────────────────
RUN chown -R ${USERNAME} /workspace

RUN echo "source /opt/ros/${ROS_DISTRO}/setup.bash" >> /home/${USERNAME}/.bashrc \
    && echo "source /workspace/ros_ws/install/setup.bash" >> /home/${USERNAME}/.bashrc \
    && echo "export GZ_VERSION=garden" >> /home/${USERNAME}/.bashrc \
    && echo "export PX4_DIR=/workspace/PX4-Autopilot" >> /home/${USERNAME}/.bashrc

ENV DEBIAN_FRONTEND=
WORKDIR /workspace
USER ${USERNAME}
CMD ["/bin/bash"]
