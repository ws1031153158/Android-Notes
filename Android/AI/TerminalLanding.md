# ML Kit
移动端机器学习 SDK，本地运行，跨平台。  
提供：  
视觉 (Vision) API：    
1.条码扫描 (Barcode Scanning)：识别各种格式的条形码和二维码。  
2.人脸检测 (Face Detection)：检测人脸、识别面部特征（如眼睛、嘴巴）。  
3.文字识别 (Text Recognition / OCR)：从图像中识别并提取文字。  
4.图像标注 (Image Labeling)：识别图像中的物体、场景和活动。  
5.对象检测与追踪 (Object Detection & Tracking)：实时检测并追踪视频帧中的物体。  
6.姿势检测 (Pose Detection)：分析人体姿态。  

自然语言 (Natural Language) API：  
1.语言识别 (Language Identification)：确定文本的语言。  
2.翻译 (Translation)：在不同语言之间翻译文本。  
3.智能回复 (Smart Reply)：根据聊天历史生成智能回复建议。  
4.实体提取 (Entity Extraction)：提取文本中的结构化数据（如日期、地址）。  
# TFLite
移动设备、嵌入式设备和物联网（IoT）边缘设备上运行机器学习模型的开源轻量级框架，本地模型推理
## 核心组成
转换器 (Converter)：将传统的 TensorFlow 模型（训练完成的）转换为 TFLite 专用的轻量级 .tflite 文件（使用 FlatBuffer 格式），这一步包含优化和量化（减小模型体积）。  
解释器 (Interpreter)：在本地端侧设备上加载 .tflite 模型，并通过优化后的内核（Kernel）执行高效推理。  
