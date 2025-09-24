# VLM study
export LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libffi.so.7:/lib/x86_64-linux-gnu/libncursesw.so.6

配置ROS2 Foxy
安装gazebo11
sudo apt install gazebo11
sudo apt install ros-foxy-gazebo-ros-pkgs
sudo apt install ros-foxy-gazebo-*

例程测试
# 终端输入启动gazebo案例
gazebo --verbose /opt/ros/foxy/share/gazebo_plugins/worlds/gazebo_ros_diff_drive_demo.world
# 打开另一个终端，快捷键是ctrl+alt+T，小车运动
ros2 topic pub /demo/cmd_demo geometry_msgs/Twist '{linear: {x: 1.0}}' -1
如果需要经常在该环境中运行，可以将库路径设置添加到环境配置中：
bash

# 编辑环境配置文件
echo 'export LD_LIBRARY_PATH=/usr/lib/x86_64-linux-gnu:/lib/x86_64-linux-gnu:$LD_LIBRARY_PATH' >> ~/.bashrc
echo 'source /opt/ros/foxy/setup.bash' >> ~/.bashrc

# 重新加载配置
source ~/.bashrc

安装PX4


要在Gazebo中搭建一个基于现有RGB图像的“较为精确”的仿真环境，这是一个非常具有挑战性但有前景的方向。这通常涉及到 三维重建 (3D Reconstruction) 和 场景语义理解 (Semantic Scene Understanding) 技术。

核心思路： 利用多视角的RGB图像，通过计算机视觉技术重建出场景的三维几何信息（点云或网格模型），然后将这些三维模型导入Gazebo。同时，可能需要对重建出的场景进行语义分割或识别，以识别出地面、建筑物、障碍物、道路等，以便在Gazebo中进行更高级的物理交互和语义查询。

下面是详细的技术路线和步骤：
技术路线：基于RGB图像在Gazebo中构建精确仿真环境
1. 数据采集与准备

    图像采集：

        多视角： 从不同角度、不同位置对目标施工或乡村地区进行密集的RGB图像采集。图像之间的重叠度要高，以利于三维重建。

        均匀覆盖： 确保整个区域被均匀覆盖，避免数据空洞。

        相机标定： 如果条件允许，使用已知内参和外参的相机，或者在采集后对相机进行标定，这将大大提高三维重建的精度。如果无法预先标定，许多三维重建工具也支持同时进行相机自标定（Bundle Adjustment）。

        辅助信息（可选但推荐）： 如果能获取到图像对应的GPS/IMU数据，可以进一步辅助三维重建的尺度恢复和地理定位。

2. 三维重建 (3D Reconstruction)

这是将2D图像转换为3D模型的关键步骤。

    SfM/MVS (Structure-from-Motion / Multi-View Stereo) 技术：

        SfM： 从无序的2D图像中同时估计相机位姿和稀疏三维点云。常用的开源库有 COLMAP、OpenMVG。

        MVS： 在SfM得到的相机位姿和稀疏点云基础上，进一步估计密集的点云或表面网格。常用的开源库有 OpenMVS、PMVS/CMVS。

        流程示例：

            特征点提取与匹配： 使用SIFT/ORB等算法在图像中提取特征点，并进行匹配。

            几何校验： 使用RANSAC等方法剔除错误的匹配点。

            SfM： 估计相机位姿和稀疏点云。

            MVS： 生成密集点云。

            网格化： 将密集点云转换为三角网格模型（例如，使用Poisson Surface Reconstruction）。

            纹理映射： 将原始RGB图像的纹理信息映射到三维模型上。

    商业软件/服务 (如果预算和资源允许)：

        Agisoft Metashape (以前的Photoscan)： 非常流行和强大的商业软件，用户友好，效果好。

        RealityCapture： 速度快，效果好，尤其适合处理大量图像。

        ContextCapture： 专业的城市级三维重建软件。

        Pix4D Mapper： 专注于无人机航测。

3. 模型后处理与优化

    模型简化 (Decimation)： SfM/MVS生成的模型通常包含非常多的三角形面片，文件体积大，可能导致Gazebo性能下降。需要使用工具（如 MeshLab、Blender）进行模型简化，减少面片数量，同时尽量保持视觉细节。

    模型清理： 移除重建过程中产生的噪声、漂浮物或不完整的几何结构。

    坐标系调整： 将模型的坐标系调整到适合Gazebo的右手坐标系，并确保原点位置合理。

    纹理优化： 合并纹理图集，压缩纹理文件，优化纹理尺寸。

    分割与标注 (可选但推荐)：

        如果需要对不同物体（如地面、建筑物、树木）进行独立的物理属性设置，或者进行VLM/VLA的语义理解，您可能需要手动或半自动地将重建出的整个大模型分割成若干个小模型，并进行语义标注。这可以通过MeshLab、Blender等工具完成。

4. 导入Gazebo

    模型格式： Gazebo支持多种3D模型格式，最常用的是 .dae (Collada) 和 .stl。带纹理的模型通常使用 .dae。

    Gazebo SDF 文件：

        为每个导入的模型创建一个SDF (Simulation Description Format) 文件。

        <model> 标签： 包含模型的路径、姿态、名称等。

        <link> 标签： 定义模型的物理属性。

        <visual> 标签： 引用纹理模型 (例如，mesh://path/to/your_model.dae)。

        <collision> 标签： 定义碰撞体。对于复杂的重建模型，通常使用简化的碰撞体（如包围盒、凸包），或直接引用视觉模型作为碰撞体（可能导致性能下降）。在精确仿真中，如果需要无人机与环境进行精确碰撞检测，可能需要更精细的碰撞模型。

        地面模型： 将重建出的地面模型作为Gazebo的静态地面。

    世界文件 (.world)：

        创建一个.world文件，用于加载所有模型。

        在其中引用您创建的SDF模型，并设置它们在世界中的初始位置和姿态。

        添加光源、物理引擎设置等。

可行性探讨与注意事项

    精度与复杂度权衡：

        “较为精确”的含义： 您需要定义“精确”的程度。是几何形状的精确还原，还是语义信息的精确识别？

        计算资源： 三维重建，特别是高密度重建，需要大量的计算资源（CPU、RAM，以及一些GPU加速）。处理大规模场景可能需要数小时到数天。

        模型规模： 重建出的模型可能非常庞大，包含数百万甚至上亿个面片。直接导入Gazebo可能导致仿真性能极低。模型简化是必不可少的。

        视觉逼真 vs. 物理精确： 高度视觉逼真的模型可能在物理碰撞方面表现不佳，或者计算量过大。您需要权衡视觉效果和物理仿真的需求。

    数据质量：

        图像质量（清晰度、曝光）、采集策略（视角、重叠度）对重建结果至关重要。

        动态物体（移动的车辆、行人）会导致重建出现伪影。

    语义信息：

        原始的三维重建结果只包含几何和纹理信息，不包含语义。

        要实现VLM/VLA对仿真环境的“理解”，您需要手动或通过额外的语义分割算法对重建模型进行标注。例如，将道路、建筑、树木、草地等区域独立出来，并赋予它们相应的语义标签。

    Gazebo性能：

        大规模、高精度的模型会显著降低Gazebo的仿真帧率。

        优化模型（简化面数、减少纹理尺寸、合理设置碰撞体）是提升性能的关键。可以考虑使用Level of Detail (LOD) 技术。

    替代方案 (如果纯RGB重建过于复杂)：

        LiDAR + RGB： 如果有条件获取激光雷达数据（点云），结合RGB图像可以获得更精确和尺度的三维模型。LiDAR提供精确的几何，RGB提供纹理。

        OSM/CAD 模型 + 纹理： 对于城市区域，可以利用OpenStreetMap (OSM) 数据生成基础的建筑模型，然后将实景RGB图像作为纹理贴到这些几何体上。对于施工区域，可能需要手动建模或使用现有的CAD图纸。

        游戏引擎 (如Unity/Unreal) + ROS： 如果对Gazebo的物理仿真精度要求不是特别高，但对渲染效果和场景编辑工具要求较高，可以考虑在Unity或Unreal中构建环境，然后通过ROS Bridge与您的算法进行通信。这些引擎在处理大规模高保真模型方面通常比Gazebo更具优势。

示例流程 (使用 COLMAP + OpenMVS)

    图像采集： 使用相机拍摄施工/乡村地区的RGB图像（例如，手机、单反、无人机）。

    COLMAP SfM：

        将图像导入COLMAP，运行 Feature Extraction -> Feature Matching -> Sparse Reconstruction。这将生成稀疏点云和相机位姿。

        导出为OpenMVS格式：colmap model_converter --input_path path/to/colmap/model --output_path path/to/openmvs/model.mvs --output_type MVS

    OpenMVS MVS：

        运行 DensifyPointCloud：从稀疏点云生成密集点云。

        运行 ReconstructMesh：从密集点云生成网格模型。

        运行 RefineMesh：优化网格。

        运行 TextureMesh：将原始图像纹理映射到网格上，并导出为 .obj 格式（通常包含 .mtl 和纹理图像文件）。

    模型后处理：

        使用 MeshLab 导入 .obj 模型。

        进行 Filters -> Remeshing, Simplification and Reconstruction -> Quadric Edge Collapse Decimation 进行模型简化（例如，将面片数量减少到20%-50%）。

        清理噪声。

        导出为 .dae 格式（确保包含纹理）。

    Gazebo集成：

        创建Gazebo模型文件夹，将 .dae 文件和纹理文件放入其中。

        编写模型的 .sdf 文件，指定视觉和碰撞模型（例如， <collision><geometry><mesh><uri>model.dae</uri></mesh></geometry></collision>，或者更简化的碰撞体）。

        编写 .world 文件，加载模型。

通过以上步骤，您可以在Gazebo中得到一个基于实景RGB图像重建出的仿真环境。这为VLM/VLA提供了更真实的视觉输入，使其能够更好地理解和交互。
