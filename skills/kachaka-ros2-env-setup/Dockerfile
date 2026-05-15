FROM osrf/ros:humble-desktop-full

ENV DEBIAN_FRONTEND=noninteractive
ENV RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
ENV ROS_LOCALHOST_ONLY=1

RUN apt-get update -qq \
    && apt-get install -y -qq --no-install-recommends \
        ros-humble-rmw-cyclonedds-cpp \
        ros-humble-teleop-twist-keyboard \
    && rm -rf /var/lib/apt/lists/*

RUN echo "source /opt/ros/humble/setup.bash" >> /root/.bashrc

CMD ["bash"]
